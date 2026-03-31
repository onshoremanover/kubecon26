# Katib

Katib is a Kubernetes-native hyperparameter tuning and neural architecture search (NAS) system, part of the Kubeflow ecosystem. It runs tuning experiments as Kubernetes custom resources, parallelizing trials across pods.

## How it works

```
Experiment (defines search space + objective)
    │
    ▼
Katib controller
    │  creates N trials based on suggestion algorithm
    ▼
Trial (maps hyperparameter set → training job)
    │  wraps a PyTorchJob / TFJob / plain Job
    ▼
Metrics collector
    │  scrapes objective metric from pod logs or Prometheus
    ▼
Suggestion service
    │  uses collected results to propose next set of hyperparameters
    └──► repeat until maxTrialCount or goal reached
```

## Experiment spec

```yaml
apiVersion: kubeflow.org/v1beta1
kind: Experiment
metadata:
  name: my-hp-tuning
  namespace: kubeflow
spec:
  objective:
    type: maximize
    goal: 0.99
    objectiveMetricName: val-accuracy

  algorithm:
    algorithmName: bayesianoptimization

  parallelTrialCount: 4
  maxTrialCount: 20
  maxFailedTrialCount: 3

  parameters:
    - name: lr
      parameterType: double
      feasibleSpace:
        min: "0.0001"
        max: "0.01"
    - name: batch-size
      parameterType: int
      feasibleSpace:
        min: "16"
        max: "128"
    - name: optimizer
      parameterType: categorical
      feasibleSpace:
        list: ["adam", "sgd", "rmsprop"]

  trialTemplate:
    primaryContainerName: training-container
    trialParameters:
      - name: learningRate
        description: Learning rate
        reference: lr
      - name: batchSize
        description: Batch size
        reference: batch-size
      - name: optimizer
        description: Optimizer
        reference: optimizer
    trialSpec:
      apiVersion: batch/v1
      kind: Job
      spec:
        template:
          spec:
            containers:
              - name: training-container
                image: my-org/my-trainer:latest
                command:
                  - python
                  - train.py
                  - "--lr=${trialParameters.learningRate}"
                  - "--batch-size=${trialParameters.batchSize}"
                  - "--optimizer=${trialParameters.optimizer}"
            restartPolicy: Never
```

## Supported algorithms

| Algorithm | Type | Notes |
|---|---|---|
| `random` | Random search | Good baseline, fully parallel |
| `grid` | Grid search | Exhaustive, use for small spaces |
| `bayesianoptimization` | Bayesian (GP) | Sample-efficient, good for expensive trials |
| `hyperband` | Multi-fidelity | Early-stops bad trials, fast |
| `tpe` | Tree-structured Parzen Estimator | Strong general-purpose algorithm |
| `cmaes` | CMA-ES | Good for continuous spaces |
| `enas` | Neural Architecture Search | NAS via ENAS |
| `darts` | Neural Architecture Search | Differentiable NAS |

## Metrics collection

Katib collects the objective metric from your training job. The simplest approach is printing to stdout in a specific format:

```python
# In your training script, print metrics each epoch:
print(f"val-accuracy={val_acc:.4f}")
```

Katib's file/stdout collector scrapes this pattern automatically. You can also use:
- **Prometheus** — collector scrapes a `/metrics` endpoint
- **TensorFlow Event** — collector reads TFEvent files
- **Custom** — implement your own collector sidecar

## Checking results

```sh
# List experiments
kubectl get experiments -n kubeflow

# Describe an experiment (shows best trial, current status)
kubectl describe experiment my-hp-tuning -n kubeflow

# List trials
kubectl get trials -n kubeflow -l experiment=my-hp-tuning

# Get the optimal parameters found
kubectl get experiment my-hp-tuning -n kubeflow -o jsonpath='{.status.currentOptimalTrial}'
```

## Python SDK

```python
from kubeflow.katib import KatibClient
from kubeflow.katib import V1beta1ExperimentSpec, V1beta1ObjectiveSpec, V1beta1AlgorithmSpec

client = KatibClient(namespace="kubeflow")

# Create experiment from a spec dict or V1beta1Experiment object
client.create_experiment(experiment, namespace="kubeflow")

# Wait for completion and get best parameters
experiment = client.wait_for_experiment_condition("my-hp-tuning")
best = client.get_optimal_hyperparameters("my-hp-tuning")
print(best)
```

## Early stopping

Katib supports early stopping to kill unpromising trials before they finish:

```yaml
earlyStopping:
  algorithmName: medianstop
  algorithmSettings:
    - name: min_trials_required
      value: "3"
    - name: start_step
      value: "5"
```

## Key points

- Trials are just Kubernetes Jobs (or Training Operator jobs) — any training framework works
- Parallelism is controlled by `parallelTrialCount`; Katib queues the rest
- Bayesian optimization and TPE are the most sample-efficient algorithms for expensive GPU training jobs
- Katib can be used standalone without the rest of Kubeflow
- Results and trial history are stored in a MySQL database deployed alongside Katib
