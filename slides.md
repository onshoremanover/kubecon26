# KubeCon Europe 2026 — Presentation Slides

---

## Slide 1: Inference Is the New Battleground

### The Cloud-Native Inference Challenge

> "Two-thirds of generative AI workloads already run on Kubernetes — but operationalizing inference at scale is the new bottleneck."

**The shift from training to inference**

- Previous years: cloud-native focused on *training* (distributing gradient descent, GPUs, checkpoints)
- 2026: inference is the center of gravity — low-latency, high-throughput, cost-efficient serving *at scale*

**What the ecosystem is building**

| Tool | What it does |
|---|---|
| **KServe** (CNCF) | Kubernetes-native model serving — canary rollouts, autoscaling, A/B testing out of the box |
| **llm-d** | Distributed inference framework donated by IBM Research, Red Hat & Google Cloud — "any model, any accelerator, any cloud" |
| **Kubernetes AI Conformance Program** | CNCF initiative to standardize inference APIs and reduce bespoke implementations |

**Why it matters**

- Inference is *stateless* — Kubernetes scheduling and autoscaling fit naturally
- But LLMs are large: model loading latency, KV-cache locality, and GPU memory fragmentation are unsolved challenges
- The community is converging on open standards rather than vendor-specific stacks

---

## Slide 2: Kubernetes for Research Computing

### Running Scientific Workloads on Kubernetes

**Kubernetes as an ML research platform — the Kubeflow stack**

```
Experiment design → Katib (hyperparameter tuning / NAS)
        ↓
Distributed training → Training Operator (PyTorchJob, TFJob)
        ↓
Model serving → KServe (InferenceService)
        ↓
Energy accounting → Kepler (per-pod power via eBPF + RAPL)
```

**Katib — automated experiment optimization**

- Runs hyperparameter searches as Kubernetes Jobs: Bayesian optimization, Hyperband, TPE, CMA-ES, NAS
- `parallelTrialCount` lets you fill idle GPU nodes automatically
- Objective metrics collected from stdout, Prometheus, or TF events — no training code changes required

**Energy-aware research computing**

- **Kepler** (CNCF Sandbox) uses eBPF to attribute CPU/DRAM energy consumption *per pod*
- Exposes Prometheus metrics → Grafana dashboards show which experiments cost the most power
- Enables carbon-aware scheduling decisions and budget enforcement via `PrometheusRule` alerts

**Key takeaway for research teams**

Kubernetes turns a bare-metal GPU cluster into a self-service research platform: experiments are reproducible CRDs, resource quotas enforce fairness between users, and energy monitoring closes the sustainability loop.

---

## Sources

- [Six Takeaways From KubeCon EU 2026 — Intuit Engineering (Medium)](https://medium.com/intuit-engineering/six-takeaways-from-kubecon-eu-2026-bb2ebbd8559e)
- [Highlights from KubeCon + CloudNativeCon Europe 2026 — Solo.io](https://www.solo.io/blog/highlights-from-kubecon-cloudnativecon-europe-2026)
- [KubeCon Europe 2026: The AI execution gap meets cloud-native reality — SiliconANGLE](https://siliconangle.com/2026/03/24/kubecon-europe-2026-ai-execution-gap-meets-cloud-native-reality/)
- [KubeCon Europe 2026: The Not-So-Unseen Engine Behind AI Innovation — Forrester](https://www.forrester.com/blogs/kubecon-europe-2026-the-not-so-unseen-engine-behind-ai-innovation/)
- [From Cloud-Native to AI-Native: what we saw at KubeCon 2026 — Moviri Consulting](https://consulting.moviri.com/kubecon-europe-2026-field-notes/)
