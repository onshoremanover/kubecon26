![Capsule](logos/capsule.png)

# Project Capsule - Multi-Tenancy Framework

[Capsule](https://capsule.clastix.io) is a Kubernetes-native multi-tenancy framework that lets platform teams offer self-service namespaces to tenants without giving them cluster-admin access.

## Core concept

Instead of one namespace per team, Capsule introduces a **Tenant** CRD that groups multiple namespaces under a single logical boundary. Tenants get full RBAC control inside their namespaces while the platform team enforces guardrails cluster-wide.

```
Cluster
├── Tenant: team-alpha  (owner: alice)
│   ├── Namespace: alpha-dev
│   ├── Namespace: alpha-staging
│   └── Namespace: alpha-prod
└── Tenant: team-beta   (owner: bob)
    ├── Namespace: beta-dev
    └── Namespace: beta-prod
```

## Architecture

- **capsule-controller-manager** — watches `Tenant` resources and reconciles RBAC, LimitRanges, NetworkPolicies, ResourceQuotas, and namespace labels automatically.
- **capsule-webhook** — validating/mutating webhook that intercepts requests from tenant users and enforces tenant-level policies (e.g. allowed storage classes, ingress classes, node selectors).
- **TenantAccessControlList** — policies defined on the `Tenant` spec that propagate to all member namespaces.

## Tenant spec example

```yaml
apiVersion: capsule.clastix.io/v1beta2
kind: Tenant
metadata:
  name: team-alpha
spec:
  owners:
    - name: alice
      kind: User
  namespaceOptions:
    quota: 5                        # max namespaces this tenant can create
  resourceQuotas:
    scope: Tenant                   # quota is shared across all namespaces
    items:
      - hard:
          requests.cpu: "10"
          requests.memory: 20Gi
  limitRanges:
    items:
      - limits:
          - type: Container
            default: { cpu: 500m, memory: 256Mi }
            defaultRequest: { cpu: 100m, memory: 64Mi }
  networkPolicies:
    items:
      - podSelector: {}
        policyTypes: [Ingress, Egress]
  ingressOptions:
    allowedClasses:
      matchLabels:
        tenant: team-alpha
  storageClasses:
    allowed: ["standard", "fast-ssd"]
```

## Key features

| Feature | Description |
|---|---|
| **Namespace self-service** | Tenant owners create namespaces; Capsule assigns them automatically |
| **Aggregated quotas** | ResourceQuota can span all tenant namespaces, not just one |
| **Policy propagation** | NetworkPolicy, LimitRange, annotations auto-applied to every namespace |
| **Allowed resource classes** | Restrict which StorageClasses, IngressClasses, PriorityClasses a tenant can use |
| **Node selector enforcement** | Pin tenant workloads to specific node pools via webhook |
| **Cordoning** | Freeze a tenant (block new resources) without deleting anything |

## Key points

- No changes to `kubectl` or existing tooling — tenants just work within their quota/policy boundaries.
- Works alongside standard Kubernetes RBAC; Capsule adds a layer on top, not a replacement.
- The webhook is the enforcement point — removing it removes all guardrails, so protect it with a `failurePolicy: Fail`.
- Compatible with ArgoCD: deploy `Tenant` objects via GitOps like any other CRD.
- **Capsule Proxy** is an optional add-on that makes cluster-scoped list calls (e.g. `kubectl get namespaces`) return only tenant-owned resources, improving the UX for tenant users.
