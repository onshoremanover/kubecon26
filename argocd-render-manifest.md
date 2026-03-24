![ArgoCD](logos/argocd.png)

# ArgoCD - Render Manifest

ArgoCD renders manifests at sync time by invoking a **config management plugin** or built-in tool (Helm, Kustomize, plain YAML) to produce the final Kubernetes resources that get applied to the cluster.

## How it works

1. **Source repository** — ArgoCD watches a Git repo (or Helm chart repo) defined in the `Application` spec.

2. **Manifest generation** — When a sync is triggered (manually or automatically), ArgoCD calls the configured tool to render the manifests:
   - **Helm**: runs `helm template` with the given values files/overrides.
   - **Kustomize**: runs `kustomize build`.
   - **Plain directory**: reads raw YAML/JSON files.
   - **Config Management Plugin (CMP)**: a sidecar in the `argocd-repo-server` pod that executes a custom `generate` command and returns rendered YAML on stdout.

3. **Diff & apply** — The rendered manifests are compared against the live cluster state. ArgoCD shows the diff in the UI and applies only the delta on sync.

4. **CMP sidecar flow**:
   ```
   argocd-repo-server
   └── CMP sidecar (plugin container)
       ├── discover  →  matches the app's source (e.g. checks for a file pattern)
       └── generate  →  stdout = rendered YAML sent back to repo-server
   ```
   The sidecar communicates with `argocd-repo-server` over a Unix socket (`/home/argocd/cmp-server/plugins/<plugin-name>/plugin.sock`).

5. **Plugin config** (`plugin.yaml` inside the sidecar image):
   ```yaml
   apiVersion: argoproj.io/v1alpha1
   kind: ConfigManagementPlugin
   metadata:
     name: my-plugin
   spec:
     discover:
       find:
         glob: "**/plugin-marker.yaml"
     generate:
       command: ["sh", "-c"]
       args: ["my-render-script.sh"]
   ```

## Key points

- Rendering always happens **server-side** in `argocd-repo-server`, not on the client.
- The rendered output is **ephemeral** — ArgoCD re-renders on every sync/refresh.
- Environment variables and parameters can be injected via the `Application` spec (`spec.source.plugin.env`).
- For Helm, `helm template` is used (not `helm install`), so no Helm release state is stored in the cluster.
- Resource tracking is done via labels/annotations injected by ArgoCD after rendering.
