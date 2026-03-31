# Kairos

Kairos is an open-source, immutable Linux meta-distribution designed for running Kubernetes at the edge. It turns any Linux base (Ubuntu, openSUSE, Alpine, etc.) into a declarative, self-upgrading OS optimized for unattended edge nodes.

## The problem with general-purpose OS at the edge

Running Raspbian (or any standard distro) on edge nodes creates operational problems at scale:

| Problem | Raspbian / standard distro | Kairos |
|---|---|---|
| **Upgrades** | `apt upgrade` mutates state; partial upgrades can brick nodes | Atomic A/B partition upgrades — old partition kept as fallback |
| **Drift** | Each node diverges over time from manual changes, cron jobs, etc. | Read-only rootfs — no in-place mutation possible |
| **Recovery** | Corrupted OS = physical intervention or re-flash | Automatic rollback to previous partition on boot failure |
| **Config management** | Requires Ansible/Chef/Puppet to maintain state across nodes | Declarative cloud-init config baked at boot time |
| **Security surface** | Full package manager, writable rootfs, many running services | Minimal read-only image; no package manager at runtime |
| **Reproductibility** | Hard to guarantee two nodes are identical | All nodes boot from the same signed OCI image |

## How it works

```
OCI image (your base OS + Kairos agent)
    │
    ▼
┌──────────────────────────────────┐
│  Disk layout                     │
│  ├── EFI / boot partition        │
│  ├── Partition A  (active)  ◄──  │  current boot
│  ├── Partition B  (passive)      │  next upgrade target
│  └── Persistent partition (rw)   │  /var, /etc overlays, data
└──────────────────────────────────┘
    │
    ▼
Kairos agent pulls new OCI image
    │  writes to passive partition
    │  sets bootloader to boot B on next reboot
    ▼
Node reboots → boots from B
    │  if healthy: B becomes active, A becomes passive
    │  if fails health check: rolls back to A automatically
```

## Declarative config (cloud-init)

Kairos uses a `cloud-config` YAML applied at first boot (and optionally on every boot via `/oem`):

```yaml
#cloud-config

hostname: edge-node-01

users:
  - name: kairos
    ssh_authorized_keys:
      - github:myusername
    groups:
      - admin

k3s:
  enabled: true
  args:
    - --disable=traefik
    - --node-label=location=warehouse-a

# Persist specific paths across upgrades
install:
  auto: true
  device: /dev/sda
  reboot: true

# Run commands on first boot
stages:
  boot:
    - name: "Configure containerd mirrors"
      files:
        - path: /etc/rancher/k3s/registries.yaml
          content: |
            mirrors:
              docker.io:
                endpoint:
                  - https://my-registry.example.com
```

## Installation methods

```sh
# Flash an SD card / USB drive
dd if=kairos-ubuntu-24.04-arm64.img of=/dev/sdX bs=4M status=progress

# Network install (PXE) — Kairos supports netboot via iPXE

# Install from a running system
kairos-agent manual-install --device /dev/sda --config cloud-config.yaml
```

## OTA upgrades

```sh
# Trigger an upgrade to a new image version
kairos-agent upgrade --image ghcr.io/kairos-io/kairos-ubuntu:v3.1.0

# Or declare it in the cloud-config for fully automatic upgrades
upgrade:
  image: ghcr.io/kairos-io/kairos-ubuntu:v3.1.0
  schedule: "0 3 * * *"   # upgrade nightly at 3am
```

## Kairos as a base OS — the layered model

Yes — Kairos is the **base OS**, and Kubernetes (k3s, k0s) runs on top of it as a managed service. Think of it like this:

```
┌─────────────────────────────────────┐
│        Your workloads (pods)        │
├─────────────────────────────────────┤
│     k3s / k0s  (Kubernetes)         │  ← runs as a systemd service
├─────────────────────────────────────┤
│     containerd  (CRI)               │  ← bundled with k3s
├─────────────────────────────────────┤
│     Kairos  (immutable Linux OS)    │  ← base OS, read-only rootfs
├─────────────────────────────────────┤
│     Hardware  (x86_64 / ARM64)      │
└─────────────────────────────────────┘
```

Kairos manages the OS lifecycle (boot, upgrades, rollback). k3s manages the Kubernetes lifecycle (API server, scheduler, kubelet). They are independent — you can upgrade the OS without touching k3s and vice versa.

## Setting up Kubernetes on edge with Kairos

### 1. Choose your image

Kairos publishes flavored images that include k3s pre-installed:

```
ghcr.io/kairos-io/kairos-ubuntu-24.04-standard-arm64-rpi4:v3.x.x
ghcr.io/kairos-io/kairos-opensuse-leap-standard-amd64:v3.x.x
```

The `standard` variant includes k3s. The `core` variant is OS-only if you want to bring your own Kubernetes distribution.

### 2. Bootstrap a single-node cluster (server)

```yaml
#cloud-config

hostname: edge-server-01

users:
  - name: kairos
    ssh_authorized_keys:
      - github:myusername

install:
  auto: true
  device: /dev/sda
  reboot: true

k3s:
  enabled: true
  args:
    - --cluster-init            # enables embedded etcd for HA later
    - --disable=traefik
    - --node-label=site=factory-a
    - --tls-san=192.168.1.10   # add node IP to TLS cert
```

Flash this config alongside the image and boot. k3s starts automatically as a systemd service on first boot.

Retrieve the kubeconfig after boot:
```sh
ssh kairos@192.168.1.10
sudo cat /etc/rancher/k3s/k3s.yaml
```

### 3. Join worker nodes

On each worker node, use a separate cloud-config that references the server:

```yaml
#cloud-config

hostname: edge-worker-01

install:
  auto: true
  device: /dev/sda
  reboot: true

k3s-agent:
  enabled: true
  env:
    K3S_URL: https://192.168.1.10:6443
    K3S_TOKEN: <token-from-server>   # cat /var/lib/rancher/k3s/server/node-token
  args:
    - --node-label=site=factory-a
    - --node-label=role=worker
```

### 4. Multi-node HA cluster with P2P networking (KubeVIP + EdgeVPN)

For truly air-gapped edge sites where nodes may not have a pre-existing overlay network, Kairos can bootstrap a cluster peer-to-peer:

```yaml
#cloud-config

hostname: edge-ha-01

p2p:
  # All nodes with the same network_token form a cluster automatically
  network_token: "{{ .NetworkToken }}"
  role: master   # or 'worker'

  vpn:
    create: true   # creates a WireGuard-based VPN between nodes

k3s:
  enabled: true
  args:
    - --cluster-init
    - --disable=traefik
```

Kairos uses **EdgeVPN** (WireGuard-based) so nodes discover and connect to each other without needing a pre-existing network or external bootstrap server.

### 5. GitOps — managing the cluster config

Once k3s is up, point a GitOps tool at the cluster from a central management plane:

```
Central cluster (e.g. your datacenter)
    │
    └── Fleet / ArgoCD
            │  watches git repo
            └── pushes manifests to edge k3s clusters
                    ├── edge-cluster-factory-a
                    ├── edge-cluster-factory-b
                    └── edge-cluster-warehouse-c
```

### 6. Upgrading the OS independently of k3s

OS and k3s upgrades are decoupled. Upgrade the OS:
```sh
kairos-agent upgrade --image ghcr.io/kairos-io/kairos-ubuntu-24.04-standard-arm64:v3.2.0
```

Upgrade k3s (via the System Upgrade Controller in-cluster):
```yaml
apiVersion: upgrade.cattle.io/v1
kind: Plan
metadata:
  name: k3s-server
spec:
  channel: https://update.k3s.io/v1-release/channels/stable
  serviceAccountName: system-upgrade
  upgrade:
    image: rancher/k3s-upgrade
  selector:
    matchLabels:
      node-role.kubernetes.io/control-plane: "true"
```

## Why not Raspbian for edge Kubernetes?

- **No atomic upgrades** — `apt upgrade` on 200 Raspberry Pis at 3am, with no rollback if half of them fail, is a disaster
- **Mutable rootfs** — one errant process or accidental `rm` can corrupt the OS permanently
- **SD card wear** — constant writes to a read-write rootfs accelerates SD card failure; Kairos's read-only rootfs dramatically reduces writes
- **Config drift** — without a config management tool, nodes diverge; Kairos enforces the declared state on every boot
- **No image-based workflow** — Raspbian doesn't fit naturally into a GitOps / OCI-image-based pipeline; Kairos images are just OCI artifacts

## Key points

- The rootfs is read-only at runtime — `/etc` and `/var` are overlay mounts backed by the persistent partition
- Custom images are built by extending official Kairos images with a standard `Dockerfile`
- Fully air-gap compatible — point upgrade images at a private OCI registry
- Works on x86_64 and ARM64 (including Raspberry Pi 4/5)
- CNCF Sandbox project
