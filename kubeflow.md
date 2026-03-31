# Kubeflow

Kubeflow is an open-source machine learning platform for Kubernetes. It provides a set of composable, portable, and scalable ML tools that cover the full ML lifecycle — from data prep and training to serving and monitoring.

## Architecture overview

```
┌─────────────────────────────────────────────────┐
│                   Kubeflow                       │
│                                                  │
│  ┌──────────┐  ┌──────────┐  ┌───────────────┐  │
│  │ Notebooks│  │Pipelines │  │ Training (TFJ/│  │
│  │ (JupyterH│  │  (KFP)   │  │ PyTorchJob..) │  │
│  └──────────┘  └──────────┘  └───────────────┘  │
│  ┌──────────┐  ┌──────────┐  ┌───────────────┐  │
│  │  Katib   │  │  KServe  │  │  Feature Store│  │
│  │(HP Tuning│  │(Serving) │  │    (Feast)    │  │
│  └──────────┘  └──────────┘  └───────────────┘  │
└─────────────────────────────────────────────────┘
             Kubernetes
```

## Core components

| Component | Purpose |
|---|---|
| **Notebooks** | JupyterHub-based notebook servers, one per user/team |
| **Pipelines (KFP)** | DAG-based ML workflow orchestration |
| **Training Operator** | Run TFJob, PyTorchJob, MXNetJob, XGBoostJob on K8s |
| **Katib** | Hyperparameter tuning and neural architecture search |
| **KServe** | Model serving (replaces KFServing) |
| **Central Dashboard** | Web UI aggregating all components |

## Kubeflow Pipelines (KFP)

Pipelines define ML workflows as Python DAGs that compile to Kubernetes resources.

```python
from kfp import dsl

@dsl.component(base_image="python:3.11")
def preprocess(data_path: str, output_path: str):
    # preprocessing logic
    ...

@dsl.component(base_image="python:3.11")
def train(data_path: str, model_path: str, epochs: int):
    # training logic
    ...

@dsl.pipeline(name="my-ml-pipeline")
def my_pipeline(data_path: str = "gs://my-bucket/data"):
    prep = preprocess(data_path=data_path, output_path="/tmp/processed")
    train(data_path=prep.output, model_path="/tmp/model", epochs=10)
```

Compile and submit:
```python
from kfp import compiler
from kfp.client import Client

compiler.Compiler().compile(my_pipeline, "pipeline.yaml")

client = Client(host="http://kubeflow.example.com/pipeline")
client.create_run_from_pipeline_func(my_pipeline, arguments={})
```

## Training Operator

Run distributed training jobs natively on Kubernetes:

```yaml
# PyTorchJob example
apiVersion: kubeflow.org/v1
kind: PyTorchJob
metadata:
  name: my-training-job
spec:
  pytorchReplicaSpecs:
    Master:
      replicas: 1
      template:
        spec:
          containers:
            - name: pytorch
              image: my-org/my-trainer:latest
              args: ["--epochs=50", "--lr=0.001"]
              resources:
                limits:
                  nvidia.com/gpu: 1
    Worker:
      replicas: 3
      template:
        spec:
          containers:
            - name: pytorch
              image: my-org/my-trainer:latest
              resources:
                limits:
                  nvidia.com/gpu: 1
```

## KServe (model serving)

```yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: my-model
spec:
  predictor:
    model:
      modelFormat:
        name: pytorch
      storageUri: gs://my-bucket/models/my-model
      resources:
        limits:
          nvidia.com/gpu: 1
```

Query the endpoint:
```sh
curl -X POST http://my-model.kubeflow.example.com/v1/models/my-model:predict \
  -H "Content-Type: application/json" \
  -d '{"instances": [[1.0, 2.0, 3.0]]}'
```

## Multi-user isolation

Kubeflow uses Kubernetes namespaces + profiles for multi-tenancy. Each user gets a `Profile` resource that creates a namespace with RBAC and resource quotas:

```yaml
apiVersion: kubeflow.org/v1
kind: Profile
metadata:
  name: alice
spec:
  owner:
    kind: User
    name: alice@example.com
  resourceQuotaSpec:
    hard:
      cpu: "16"
      memory: 64Gi
      nvidia.com/gpu: "2"
```

## Key points

- Kubeflow is modular — you can install only the components you need
- Pipelines use Argo Workflows under the hood
- Notebook servers are per-user and can request GPUs directly
- KServe supports canary rollouts, A/B testing, and autoscaling out of the box
- The Training Operator handles fault tolerance and pod restarts for distributed jobs
