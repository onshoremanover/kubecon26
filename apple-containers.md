![Apple Containers](logos/apple-containers.png)

# Apple Containers

Apple Containers is a native macOS containerization tool introduced by Apple that runs Linux containers directly on Apple Silicon using lightweight virtualization. Unlike Docker Desktop, it uses Apple's native `Virtualization.framework` and does not require a separate VM daemon.

## How it works

Each container runs in its own lightweight Linux VM using `Virtualization.framework`. There is no shared daemon — containers are isolated processes managed directly by the `container` CLI.

```
macOS (Apple Silicon)
└── Virtualization.framework
    ├── Container A  (own Linux kernel)
    ├── Container B  (own Linux kernel)
    └── Container C  (own Linux kernel)
```

## Basic usage

### Pull an image
```sh
container pull ubuntu:24.04
container pull ghcr.io/my-org/my-app:latest
```

### Run a container
```sh
# Run interactively
container run --interactive --tty ubuntu:24.04 /bin/bash

# Run in the background
container run --detach --name my-app my-image:latest

# Run with port mapping
container run --detach --publish 8080:80 nginx:latest

# Run with a volume mount
container run --volume /host/path:/container/path my-image:latest
```

### List containers
```sh
container list           # running containers
container list --all     # all containers including stopped
```

### Stop and remove
```sh
container stop my-app
container rm my-app
container rm --force my-app   # stop and remove in one step
```

### Execute a command in a running container
```sh
container exec --interactive --tty my-app /bin/bash
```

### View logs
```sh
container logs my-app
container logs --follow my-app
```

## Images

```sh
container images          # list local images
container rmi my-image    # remove an image
container build --tag my-app:latest .   # build from a Dockerfile
```

## Networking

Containers get their own network namespace. Port mapping (`--publish`) exposes ports on the macOS host.

```sh
# Expose container port 3000 on host port 3000
container run --publish 3000:3000 my-node-app:latest
```

## Volumes

```sh
# Mount a host directory
container run --volume $(pwd)/data:/app/data my-image

# Named volumes (managed by container runtime)
container volume create my-volume
container run --volume my-volume:/data my-image
container volume list
container volume rm my-volume
```

## Building images

Uses standard `Dockerfile` syntax:

```sh
container build --tag my-app:latest .
container build --tag my-app:latest --file Dockerfile.prod .
```

## Key differences from Docker

| | Apple Containers | Docker Desktop |
|---|---|---|
| **Runtime** | Virtualization.framework (native) | Docker daemon + VM |
| **Daemon** | None — each container is its own VM | Shared `dockerd` daemon |
| **Architecture** | Apple Silicon only | Intel + Apple Silicon |
| **Isolation** | Kernel-level (separate VM per container) | Shared kernel via namespace |
| **OCI compatible** | Yes | Yes |
| **Compose support** | Limited / in progress | Full Docker Compose |

## Key points

- Only supported on **Apple Silicon** (M1/M2/M3/M4).
- Images are OCI-compatible — you can use images built for `linux/arm64`.
- No background daemon means no `dockerd` to keep running; containers start their own lightweight VM.
- `amd64` images can run via Rosetta emulation but with a performance cost.
- The CLI is `container`, not `docker` — most flags follow the same conventions.
