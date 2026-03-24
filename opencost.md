![OpenCost](logos/opencost.png)

# OpenCost

[OpenCost](https://www.opencost.io) is a CNCF project (sandbox → incubating) that provides real-time cost monitoring and allocation for Kubernetes workloads. It breaks down cloud infrastructure spend by namespace, deployment, pod, label, and team.

## Core concept

OpenCost scrapes Kubernetes resource usage (CPU, memory, storage, network) and maps it against cloud provider pricing to give you a dollar cost per workload. It runs entirely in-cluster and exposes cost data via an API and UI.

```
Kubernetes Cluster
├── OpenCost (in-cluster)
│   ├── scrapes kube-state-metrics + cAdvisor
│   ├── queries cloud provider pricing APIs
│   └── exposes /allocation API + UI
└── Your workloads
    ├── namespace: team-alpha  →  $12.40/day
    ├── namespace: team-beta   →  $8.10/day
    └── namespace: infra       →  $31.00/day
```

## Architecture

- **OpenCost pod** — core engine, runs as a Deployment in the `opencost` namespace. Computes cost allocation from metrics.
- **kube-state-metrics** — required dependency, provides resource request/limit data per pod.
- **Cloud pricing integration** — queries AWS, GCP, or Azure pricing APIs to get node costs. Falls back to default prices if no cloud provider is configured.
- **Prometheus** (optional but recommended) — OpenCost exposes metrics that Prometheus scrapes; Grafana dashboards can then visualize cost over time.
- **OpenCost UI** — lightweight frontend bundled with the deployment for exploring costs interactively.

## Install (Helm)

```sh
helm repo add opencost https://opencost.github.io/opencost-helm-chart
helm repo update
helm install opencost opencost/opencost --namespace opencost --create-namespace
```

Access the UI:
```sh
kubectl port-forward -n opencost service/opencost 9090:9090
# open http://localhost:9090
```

## Cost allocation model

OpenCost allocates costs based on **resource requests** (not actual usage) by default, which reflects what is reserved on the node:

| Resource | How it's priced |
|---|---|
| CPU | Node vCPU cost × (requested CPUs / total node CPUs) |
| Memory | Node RAM cost × (requested memory / total node memory) |
| Storage | PVC cost from cloud provider disk pricing |
| Network | Egress cost (if cloud provider data available) |
| Idle | Unallocated node capacity — can be distributed or shown separately |

## Querying the API

```sh
# Cost by namespace for the last 7 days
curl "http://localhost:9003/allocation?window=7d&aggregate=namespace"

# Cost by deployment
curl "http://localhost:9003/allocation?window=1d&aggregate=deployment"

# Cost by label (e.g. team label)
curl "http://localhost:9003/allocation?window=30d&aggregate=label:team"
```

## Prometheus metrics (sample)

```
# Cost per namespace per hour
opencost_namespace_cost_hourly{namespace="team-alpha"} 0.52

# CPU cost per pod
opencost_pod_cpu_cost_hourly{pod="api-server-xyz", namespace="team-alpha"} 0.12
```

## Cloud provider setup

To get accurate pricing, point OpenCost at your cloud provider:

```yaml
# values.yaml for Helm
opencost:
  cloudProviderApiKey: ""   # for GCP
  aws:
    spot_label: "eks.amazonaws.com/capacityType"
    spot_label_value: "SPOT"
```

For on-prem or bare-metal (e.g. microk8s), you set a **custom price sheet**:
```yaml
opencost:
  customPricesEnabled: true
  defaultVCPUPrice: "0.031611"
  defaultRAMPrice: "0.004237"
  defaultStoragePrice: "0.00005479"
```

## Key features

| Feature | Description |
|---|---|
| **Real-time allocation** | Cost broken down per pod/namespace/deployment live |
| **Idle cost tracking** | Shows how much of your node spend is wasted |
| **Multi-cloud** | AWS, GCP, Azure, and on-prem custom pricing |
| **Label-based chargeback** | Aggregate costs by any Kubernetes label (team, env, etc.) |
| **External costs** | Can include costs outside the cluster (S3, BigQuery, etc.) |
| **Kubecost compatibility** | OpenCost is the open-source core that Kubecost is built on |

## Key points

- OpenCost is the **open-source core of Kubecost** — Kubecost is the commercial product built on top with extra features (alerts, budgets, SAML SSO).
- Works well with **Capsule** multi-tenancy — you can aggregate costs per Tenant by aligning Capsule tenant labels with OpenCost label aggregation.
- On a **microk8s** or bare-metal cluster, use custom pricing to get meaningful numbers even without a cloud provider.
- Idle cost is often the most actionable insight — it shows how much capacity is reserved but never used.
- The `/allocation` API is the main integration point for building custom dashboards or feeding cost data into internal tooling.
