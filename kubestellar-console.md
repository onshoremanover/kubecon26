![KubeStellar](logos/kubestellar.png)

# KubeStellar Console

KubeStellar is an open-source project that enables multi-cluster Kubernetes workload management across heterogeneous environments (edge, on-prem, cloud). The Console is its web UI for managing and observing distributed workloads.

## Core concept

KubeStellar introduces a **control plane** that sits above multiple Kubernetes clusters and lets you define where workloads should run using policy objects. The Console provides visibility and management across all registered clusters from a single pane of glass.

```
KubeStellar Control Plane
├── WDS (Workload Description Space)  ←  where you define workloads & policies
│   ├── BindingPolicy
│   └── Workload objects (Deployments, Services, etc.)
└── ITS (Inventory & Transport Space)  ←  cluster inventory & status sync
    ├── Cluster A (edge)
    ├── Cluster B (on-prem)
    └── Cluster C (cloud)
```

## Architecture

- **WDS (Workload Description Space)** — a Kubernetes API space where users submit workloads and `BindingPolicy` objects. KubeStellar propagates these to matching clusters.
- **ITS (Inventory & Transport Space)** — tracks registered clusters and transports objects between the control plane and workload clusters using `SyncerConfig`.
- **KubeStellar Controller** — reconciles `BindingPolicy` objects and drives workload distribution.
- **Syncer** — a lightweight agent running in each workload cluster that pulls down objects from the ITS and reports status back.
- **Console** — a web UI layered on top, providing cluster topology views, workload status, and policy management.

## BindingPolicy example

```yaml
apiVersion: control.kubestellar.io/v1alpha1
kind: BindingPolicy
metadata:
  name: deploy-nginx-to-edge
spec:
  clusterSelectors:
    - matchLabels:
        location: edge
  downsync:
    - objectSelectors:
        - matchLabels:
            app: nginx
```

This propagates all objects labelled `app: nginx` to every cluster labelled `location: edge`.

## Console features

| Feature | Description |
|---|---|
| **Cluster inventory** | View all registered clusters, their status, and labels |
| **Workload overview** | See where each workload is running and its rollout status |
| **BindingPolicy editor** | Create and edit propagation policies via UI |
| **Status aggregation** | Aggregated health view across all target clusters |
| **Multi-space navigation** | Switch between WDS and ITS views |

## Key points

- KubeStellar does **not** require a mesh or VPN between clusters — the Syncer in each cluster pulls from the ITS over standard HTTPS.
- Clusters can be any conformant Kubernetes distribution (k3s, OpenShift, EKS, etc.).
- `BindingPolicy` uses label selectors, making cluster targeting dynamic — add a label to a cluster and it automatically receives matching workloads.
- The Console is optional; everything can be managed via `kubectl` against the WDS.
- Designed for **edge/multi-cloud** scenarios where clusters may be intermittently connected.
