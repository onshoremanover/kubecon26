# ContainerSSH

ContainerSSH is an open-source SSH server that dynamically launches containers when users connect via SSH. Instead of logging into a static VM or host, each SSH session spins up a fresh container (in Docker, Kubernetes, etc.) and destroys it when the session ends.

## How it works

```
User (ssh user@host)
    │
    ▼
ContainerSSH server
    │  authenticates via webhook / password / pubkey
    │  asks config webhook: "what container should this user get?"
    ▼
Kubernetes / Docker backend
    │  creates pod / container for the session
    ▼
User gets a shell inside the container
    │
    └── on disconnect → container is destroyed
```

## Key components

- **SSH server** — handles inbound SSH connections, authentication, and terminal negotiation
- **Auth webhook** — ContainerSSH calls your HTTP endpoint to authenticate the user
- **Config webhook** — your HTTP endpoint returns the container spec (image, env vars, resource limits, etc.) per user/session
- **Backend** — Kubernetes, Docker, or containerd; ContainerSSH creates and manages the container lifecycle

## Basic configuration

```yaml
# config.yaml
ssh:
  hostkeys:
    - /etc/containerssh/host.key

auth:
  webhook:
    url: http://my-auth-service/auth

configserver:
  url: http://my-config-service/config

backend: kubernetes

kubernetes:
  connection:
    host: https://kubernetes.default.svc
    cacertFile: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    bearerTokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token
  pod:
    namespace: containerssh-guests
    spec:
      containers:
        - name: shell
          image: ubuntu:24.04
```

## Auth webhook

ContainerSSH POSTs to your auth endpoint and expects a JSON response:

```json
// Request
{ "username": "alice", "remoteAddress": "1.2.3.4:12345", "connectionId": "abc123", "passwordBase64": "..." }

// Response
{ "success": true }
```

## Config webhook

Your config endpoint can return a per-user container spec:

```json
// Response
{
  "backend": "kubernetes",
  "kubernetes": {
    "pod": {
      "namespace": "containerssh-guests",
      "spec": {
        "containers": [{ "name": "shell", "image": "my-org/user-env:latest" }]
      }
    }
  }
}
```

## Kubernetes deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: containerssh
spec:
  replicas: 1
  selector:
    matchLabels:
      app: containerssh
  template:
    spec:
      containers:
        - name: containerssh
          image: containerssh/containerssh:latest
          ports:
            - containerPort: 2222
          volumeMounts:
            - name: config
              mountPath: /etc/containerssh
      volumes:
        - name: config
          configMap:
            name: containerssh-config
---
apiVersion: v1
kind: Service
metadata:
  name: containerssh
spec:
  type: LoadBalancer
  ports:
    - port: 22
      targetPort: 2222
  selector:
    app: containerssh
```

## Use cases

- **Ephemeral dev environments** — each SSH session gets a fresh, pre-configured container
- **Secure bastion / jump host** — users never touch the underlying node; containers are isolated and disposable
- **Multi-tenant access** — config webhook tailors the container image/resources per user or team
- **CTF / training platforms** — spin up challenge environments on demand per participant

## Key points

- Stateless by design — containers are ephemeral; nothing persists between sessions unless you mount a PVC
- Auth and config are fully delegated to your own HTTP services, so you can integrate any identity provider
- Supports standard SSH features: port forwarding, SCP/SFTP, pseudo-TTY
- RBAC in Kubernetes controls what ContainerSSH can create — namespace it appropriately
- Works with existing SSH clients — no special client needed
