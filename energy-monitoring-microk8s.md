# Energy Monitoring on Ubuntu with MicroK8s, Prometheus and Grafana

Monitoring energy consumption of a single Ubuntu node running MicroK8s, using Prometheus and Grafana deployed on the same node. The key tool is **Kepler** (Kubernetes Efficient Power Level Exporter) — a CNCF project that uses eBPF and Linux RAPL (Running Average Power Limit) interfaces to expose per-pod and per-node energy metrics to Prometheus.

## How it works

```
Ubuntu node (bare metal or VM with RAPL access)
│
├── MicroK8s
│   ├── Prometheus        ← scrapes metrics
│   ├── Grafana           ← visualises metrics
│   └── Kepler (DaemonSet)
│       ├── reads /sys/class/powercap/intel-rapl  (CPU/DRAM energy)
│       ├── reads perf_events via eBPF            (per-process attribution)
│       └── exposes /metrics on port 9102
│
└── Hardware energy counters (RAPL)
    ├── package  (CPU socket total)
    ├── core     (CPU cores)
    ├── uncore   (GPU/iGPU on-die)
    └── dram     (memory subsystem)
```

Kepler reads hardware energy counters and uses eBPF to attribute consumption to individual processes and Kubernetes pods, then exposes everything as Prometheus metrics.

> **Note:** RAPL is available on Intel (Sandy Bridge+) and AMD (Zen+) CPUs. On VMs, RAPL access depends on whether the hypervisor exposes the MSRs — works on bare metal and some cloud instances (e.g. AWS metal instances). On Raspberry Pi / ARM, Kepler falls back to software estimation models.

## 1. Enable MicroK8s addons

```sh
# Install MicroK8s if not already present
sudo snap install microk8s --classic

sudo microk8s enable dns
sudo microk8s enable prometheus   # deploys kube-prometheus-stack (Prometheus + Grafana + Alertmanager)
sudo microk8s enable hostpath-storage
```

The `prometheus` addon deploys the full `kube-prometheus-stack` into the `monitoring` namespace.

Check it's running:
```sh
sudo microk8s kubectl get pods -n monitoring
```

## 2. Enable RAPL access on the host

Kepler needs access to Linux power cap interfaces. On Ubuntu, ensure the kernel module is loaded:

```sh
# Load the RAPL kernel module
sudo modprobe intel_rapl_common   # Intel
# or
sudo modprobe amd_energy          # AMD

# Verify RAPL is exposed
ls /sys/class/powercap/
# should show: intel-rapl  intel-rapl:0  intel-rapl:0:0  etc.

# Make it persistent across reboots
echo "intel_rapl_common" | sudo tee /etc/modules-load.d/rapl.conf
```

## 3. Deploy Kepler

```sh
# Add the Kepler Helm chart
sudo microk8s helm3 repo add kepler https://sustainable-computing-io.github.io/kepler-helm-chart
sudo microk8s helm3 repo update

# Install Kepler into the monitoring namespace
sudo microk8s helm3 install kepler kepler/kepler \
  --namespace monitoring \
  --set serviceMonitor.enabled=true \
  --set serviceMonitor.labels.release=prometheus \
  --set extraEnvVars.ENABLE_EBPF_CGROUPID=true
```

The `serviceMonitor.labels.release=prometheus` label must match the label selector used by the Prometheus operator deployed by MicroK8s. Verify the label if scraping doesn't start:

```sh
sudo microk8s kubectl get prometheus -n monitoring -o jsonpath='{.items[0].spec.serviceMonitorSelector}'
```

Check Kepler is running:
```sh
sudo microk8s kubectl get pods -n monitoring -l app.kubernetes.io/name=kepler
sudo microk8s kubectl logs -n monitoring -l app.kubernetes.io/name=kepler
```

Verify metrics are being exposed:
```sh
# Port-forward to check raw metrics
sudo microk8s kubectl port-forward -n monitoring svc/kepler 9102:9102 &
curl http://localhost:9102/metrics | grep kepler_node
```

## 4. Key Kepler metrics

| Metric | Description |
|---|---|
| `kepler_node_package_joules_total` | Total CPU package energy (Joules), cumulative counter |
| `kepler_node_dram_joules_total` | Total DRAM energy (Joules), cumulative counter |
| `kepler_node_core_joules_total` | CPU core energy (Joules) |
| `kepler_node_uncore_joules_total` | Uncore (iGPU, cache) energy |
| `kepler_container_package_joules_total` | Per-container CPU package energy |
| `kepler_container_dram_joules_total` | Per-container DRAM energy |
| `kepler_node_platform_joules_total` | Whole-platform energy (if ACPI/BMC available) |

Convert from cumulative Joules to instantaneous Watts in PromQL:
```promql
rate(kepler_node_package_joules_total[1m])   -- Watts (J/s) averaged over 1 minute
```

## 5. Grafana dashboards

### Access Grafana

```sh
# Get the Grafana service
sudo microk8s kubectl get svc -n monitoring | grep grafana

# Port-forward to access locally
sudo microk8s kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80

# Default credentials (set in the prometheus addon)
# user: admin  password: prom-operator
# or check: kubectl get secret -n monitoring kube-prometheus-stack-grafana -o jsonpath='{.data.admin-password}' | base64 -d
```

### Import the Kepler dashboard

Kepler provides an official Grafana dashboard (ID `15759`):

1. Open Grafana → **Dashboards** → **Import**
2. Enter dashboard ID `15759` and click **Load**
3. Select your Prometheus datasource and click **Import**

### Useful PromQL queries for custom panels

```promql
# Node total power draw (Watts)
rate(kepler_node_package_joules_total[1m]) + rate(kepler_node_dram_joules_total[1m])

# Power per namespace (top consumers)
topk(10,
  sum by (namespace) (
    rate(kepler_container_package_joules_total[1m])
  )
)

# Power per pod
sum by (pod_name, namespace) (
  rate(kepler_container_package_joules_total[1m]) +
  rate(kepler_container_dram_joules_total[1m])
)

# Energy consumed in the last hour (Wh) per namespace
sum by (namespace) (
  increase(kepler_container_package_joules_total[1h])
) / 3600

# CO2 estimate — multiply Wh by your grid carbon intensity (e.g. 0.233 kgCO2/kWh for EU average)
(
  sum(increase(kepler_node_package_joules_total[1h])) / 3600 / 1000
) * 0.233
```

## 6. node_exporter hardware metrics (complement to Kepler)

The `prometheus` MicroK8s addon already deploys `node_exporter`. Useful complementary metrics:

```promql
# CPU utilisation
100 - (avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100)

# CPU temperature (if sensors available)
node_hwmon_temp_celsius

# Fan speed
node_hwmon_fan_rpm
```

## 7. Alerting on power budget

```yaml
# Add to a PrometheusRule resource
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: energy-alerts
  namespace: monitoring
  labels:
    release: prometheus
spec:
  groups:
    - name: energy
      rules:
        - alert: NodeHighPowerDraw
          expr: |
            rate(kepler_node_package_joules_total[5m])
            + rate(kepler_node_dram_joules_total[5m]) > 150
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "Node power draw exceeds 150W"
            description: "Current draw: {{ $value | humanize }}W"
```

Apply it:
```sh
sudo microk8s kubectl apply -f energy-alerts.yaml
```

## Key points

- Kepler is the de-facto standard for Kubernetes energy observability — CNCF Sandbox project
- RAPL gives you CPU + DRAM energy; whole-platform power (including NIC, disk, PSU losses) requires ACPI or a BMC/IPMI interface
- On ARM (Raspberry Pi), RAPL is not available — Kepler uses an estimation model based on CPU utilisation which is less accurate
- Energy metrics are cumulative counters (Joules) — always use `rate()` or `increase()` in PromQL to get power (Watts) or energy over a window
- The `serviceMonitor` label must match the Prometheus operator's `serviceMonitorSelector` or scraping won't start — this is the most common setup issue
