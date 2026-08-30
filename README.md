# 🏠 Fedora CoreOS Homelab Server

My private, declarative homelab environment powered by **Fedora CoreOS**, **k3s**, and **ArgoCD**. This project aims to provide a lightweight, secure, and reliable infrastructure for self-hosted services, fully managed through GitOps — everything running in the cluster is defined and version-controlled in this repository.

## 🛠 Tech Stack

| Component        | Tool / Technology            |
|-------------------|-------------------------------|
| OS                | Fedora CoreOS (immutable, declarative) |
| Orchestration     | k3s (lightweight Kubernetes)  |
| GitOps / CD       | ArgoCD                        |
| Ingress / Proxy   | Traefik (bundled with k3s)    |
| Load Balancing    | ServiceLB (bundled with k3s)  |

## 🖥 Hardware

| Component | Spec                  |
|-----------|------------------------|
| CPU       | AMD Ryzen H255         |
| RAM       | 16 GB DDR5             |
| iGPU      | AMD Radeon 780M        |
| LAN       | 2.5 Gbps                |

## 🔄 GitOps Workflow

This repository is the **single source of truth** for everything deployed on the cluster. There are only ever two manual `kubectl apply` commands in this project's entire lifecycle — bootstrapping the two root applications below. Everything after that is fully automated through the **app-of-apps** pattern:

```
Edit manifests locally
        ↓
git commit & push to GitHub
        ↓
ArgoCD detects the change (auto-sync, ~3 min or on webhook)
        ↓
ArgoCD applies changes to the cluster
        ↓
Self-heal: any manual drift in the cluster is reverted to match Git
```

Sync policy for all apps: `automated: prune + selfHeal`, meaning:
- Resources removed from Git are pruned from the cluster.
- Any manual changes made directly in the cluster (e.g. `kubectl scale`) are automatically reverted to match the state defined in Git.

### Dual app-of-apps: user apps vs. infrastructure

Two independent root `Application` resources split responsibilities cleanly:

| Root Application | Watches path            | Purpose                                                        |
|-------------------|--------------------------|-----------------------------------------------------------------|
| `root`            | `apps/`                 | User-facing services (podinfo, and future apps like Pi-hole, Unbound, Home Assistant). |
| `root-infra`      | `argocd/applications/`  | Cluster infrastructure components (e.g. Sealed Secrets, future cert-manager, monitoring). |

Both are bootstrapped once, manually:

```bash
sudo kubectl apply -f root-app.yaml
sudo kubectl apply -f root-infra.yaml
```

From that point on, adding a new user app or a new infrastructure component is **just a `git push`** — no further manual `kubectl apply` is ever needed for that component.

## 📁 Repository Structure

```
.
├── root-app.yaml               # bootstraps root: watches apps/
├── root-infra.yaml             # bootstraps root-infra: watches argocd/applications/
├── .gitignore                  # excludes raw secrets and kubeseal temp files
├── apps/
│   ├── podinfo/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── kustomization.yaml
│   ├── pihole/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── pvc.yaml
│   │   ├── sealed-secret.yaml   # encrypted WEBPASSWORD, safe to commit
│   │   └── kustomization.yaml
│   ├── llamacpp/
│   │   ├── deployment.yaml       # llama.cpp server-vulkan, GPU offload via /dev/dri
│   │   ├── service.yaml
│   │   ├── pvc.yaml              # model storage
│   │   ├── ingress.yaml          # exposed at http://ai.home via Traefik
│   │   └── kustomization.yaml
│   ├── homepage/
│   │   ├── configmap.yaml        # settings/services/widgets, config lives entirely in Git
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── ingress.yaml          # exposed at http://panel.home
│   │   └── kustomization.yaml
│   └── argocd-ingress/
│       ├── ingress.yaml          # exposed at http://argocd.home
│       └── kustomization.yaml
└── argocd/
    └── applications/
        ├── podinfo-app.yaml     # Application CR pointing at apps/podinfo
        ├── pihole-app.yaml      # Application CR pointing at apps/pihole
        ├── llamacpp-app.yaml    # Application CR pointing at apps/llamacpp
        ├── homepage-app.yaml    # Application CR pointing at apps/homepage
        ├── argocd-ingress-app.yaml # Application CR pointing at apps/argocd-ingress
        └── sealed-secrets.yaml  # Application CR (Helm chart, infra component)
```

- `root-app.yaml` / `root-infra.yaml` — the only two manifests ever applied manually to the cluster.
- `apps/` — Kubernetes manifests for user-facing applications, one folder per app.
- `argocd/applications/` — ArgoCD `Application` CRs for both user apps and infrastructure components, watched exclusively by `root-infra`.

## 🚀 Deployed Apps

| App     | Description                        | Namespace | Status         |
|---------|-------------------------------------|-----------|----------------|
| podinfo | GitOps flow test / demo app         | `apps`    | ✅ Running (LoadBalancer, port 9898) |
| pihole  | Network-wide ad/tracker blocking, DNS server for the LAN | `apps` | ✅ Running (LoadBalancer — DNS on port 53, web UI on port 8080) |
| llamacpp | Local LLM inference (Gemma 4 E4B Uncensored) with GPU acceleration | `apps` | ✅ Running (Ingress — `http://ai.home`) |
| homepage | Dashboard / landing page for the whole homelab | `apps` | ✅ Running (Ingress — `http://panel.home`) |

## 🧩 Infrastructure Components

| Component       | Description                                  | Namespace     | Status    |
|------------------|-----------------------------------------------|----------------|-----------|
| sealed-secrets   | Encrypts secrets so they're safe to commit to this public repo | `kube-system` | ✅ Running |

## 🔐 Secrets Management

Since this repository is **public**, plaintext Kubernetes `Secret` objects can never be committed to it. Secrets are encrypted client-side with [Sealed Secrets](https://github.com/bitnami/sealed-secrets) before they ever touch Git — only the `sealed-secrets-controller` running in the cluster holds the private key needed to decrypt them.

Workflow for adding a secret to any app (Pi-hole, Unbound, Home Assistant, etc.):

```bash
# 1. Create the raw secret locally — never committed
kubectl create secret generic <app>-secret \
  --namespace=apps \
  --from-literal=KEY='value' \
  --dry-run=client -o yaml > secret-tmp.yaml

# 2. Encrypt it with kubeseal
kubeseal \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system \
  --scope namespace-wide \
  -f secret-tmp.yaml \
  -w apps/<app>/sealed-secret.yaml

# 3. Delete the raw file, commit only the encrypted one
rm secret-tmp.yaml
git add apps/<app>/sealed-secret.yaml
git commit -m "Add sealed secret for <app>"
git push
```

The resulting `SealedSecret` is safe to publish — it can only be decrypted by the controller running on this specific cluster.

> ⚠️ The controller's private key is backed up **outside** of Git (password manager / encrypted storage). Losing it without a backup means losing the ability to decrypt any existing `SealedSecret` after a cluster rebuild.

## 🕳️ Pi-hole Notes

A few deployment decisions worth documenting for anyone reproducing this setup:

- **Persistent storage**: `/etc/pihole` and `/etc/dnsmasq.d` are backed by a PVC (`local-path` StorageClass), so config, block lists, and query logs survive pod restarts.
- **Port 80 conflict**: since Traefik's `svclb` DaemonSet already binds hostPort 80 on this single-node cluster, Pi-hole's web UI is exposed on port **8080** instead of 80 to avoid a scheduling conflict on the `LoadBalancer` Service.
- **Pi-hole v6 env vars**: the `pihole/pihole` image moved to CalVer versioning with v6, which renamed most environment variables (e.g. `WEBPASSWORD` → `FTLCONF_webserver_api_password`). This deployment uses the new naming.
- **Upstream DNS**: configured with Quad9 (`9.9.9.9`) as the upstream resolver. This will be replaced by a local Unbound instance for full recursive resolution once that's deployed.
- **Password**: set via a `SealedSecret` following the workflow described in [Secrets Management](#-secrets-management) above.

## 🤖 Local LLM Inference Notes

Runs [llama.cpp](https://github.com/ggml-org/llama.cpp) with GPU acceleration on the Radeon 780M iGPU via the **Vulkan** backend — chosen over ROCm since Vulkan support for this GPU (RADV/Mesa) is mature and requires no unofficial workarounds, unlike the still-experimental ROCm support for gfx1103.

A few things worth documenting for anyone reproducing this:

- **GPU passthrough**: `/dev/dri` is passed into the container via `hostPath`, with the container running `privileged: true`. This is the simplest working setup; scoping it down to `supplementalGroups` with the host's `render` group GID instead of full `privileged` is a good future hardening step.
- **Known bug in the official image**: `ghcr.io/ggml-org/llama.cpp:server-vulkan` fails to detect any Vulkan devices ([upstream issue](https://github.com/ggml-org/llama.cpp/issues/24651)), even though the exact same Vulkan binary works when run manually inside that same container. This deployment uses the community-maintained `ghcr.io/kth8/llama-server-vulkan` image instead, which doesn't have this issue.
- **Shared memory budget**: the 780M uses UMA (no dedicated VRAM) — available GPU memory is whatever the BIOS allocates from system RAM, in this case only a few GB free after driver overhead (`--list-devices` reports ~8GB total, ~1GB free at idle). Qwen3.5-9B (~5.6GB) didn't fully fit and silently fell back to partial CPU inference (~7 tok/s). **Gemma 4 E2B** (~3.1GB) fit comfortably at ~23 tok/s. Currently running **Gemma 4 E4B Uncensored (HauhauCS)** at ~5.3GB, which lands right at the edge of the budget — currently getting ~13 tok/s with no CPU load during inference, confirming it's running on GPU despite being close to the memory ceiling.
- **Model downloads**: the model file is fetched by an `initContainer` (not baked into the image) so swapping models is just a one-line change + `git push`, no rebuild. The download is atomic (`.tmp` file + `mv` on success) to avoid loading a corrupted/partial model after an interrupted pod restart — an early version without this caused a silent failure that took a while to diagnose.
- **Monitoring init container progress on large downloads**: `kubectl logs -f` on an init container can appear to hang on large file downloads — `curl`'s progress bar updates via carriage return (`\r`), which doesn't always stream correctly through Kubernetes' log API. `kubectl exec <pod> -c <initcontainer> -- stat -c%s /path/to/file.tmp`, checked twice a few seconds apart, is a more reliable way to confirm a download is actually progressing.
- **Access**: exposed via Traefik `Ingress` at `http://ai.home` (a local DNS record added in Pi-hole), rather than a `LoadBalancer` port, to avoid consuming another dedicated IP/port and to avoid a repeat of the port 80 conflict seen with Pi-hole's web UI. Note: `.local` was tried first and didn't work — it's a reserved mDNS domain that most systems intercept before it ever reaches a normal DNS resolver.
- **API**: OpenAI-compatible, served on the same port as the built-in web chat UI (`/v1/chat/completions`, etc.) — useful for wiring up future services like n8n.

## 📊 Homepage Dashboard Notes

[Homepage](https://gethomepage.dev) was chosen over alternatives like Homarr specifically because its entire configuration is plain YAML files (`settings.yaml`, `services.yaml`, `widgets.yaml`) rather than state stored in a database inside the container — this keeps it consistent with the GitOps principle used everywhere else in this repo: the config lives in a `ConfigMap`, mounted into the pod, fully version-controlled.

- **ConfigMap changes require a manual pod restart**: updating the `ConfigMap` in Git and letting ArgoCD sync it updates the file on disk inside the pod (with a short kubelet propagation delay), but Homepage doesn't watch for changes and reload automatically. After any `configmap.yaml` edit, run `kubectl rollout restart deployment homepage -n apps` to pick it up. A more robust future fix would be a checksum annotation on the pod template (via Kustomize's `configMapGenerator`) to trigger this automatically.
- **Pi-hole v6 widget syntax**: requires an explicit `version: 6` field and the API key/password passed via `key`, since Pi-hole v6 replaced the old `/admin/api.php` token-based API — the widget needs the *port* included in its `url` (e.g. `:8080`) when it differs from the service's default, which isn't obvious from the docs and initially caused silent connection timeouts.
- **Reusing an existing secret**: rather than creating a second `SealedSecret` with a duplicate password, the Pi-hole widget's API key references the *same* `pihole-secret` / `WEBPASSWORD` key already used by the Pi-hole deployment itself — safe because that secret was sealed with `--scope namespace-wide`, and it means a future password rotation only has to happen in one place.

## ⚠️ Known Issue: ArgoCD UI Ingress redirects to HTTPS

An `Ingress` for the ArgoCD UI (`http://argocd.home`) was added but currently returns a `307` redirect to HTTPS with no valid cert behind it — `argocd-server` runs with TLS enabled by default, and the fix (`server.insecure: "true"` in `argocd-cmd-params-cm`) hasn't been applied yet. Tracked as a follow-up; for now the ArgoCD UI is reached via `kubectl port-forward svc/argocd-server -n argocd 8080:443`.

## 🌐 DNS Architecture Note: the host does not use Pi-hole as its resolver

Early on, the k3s **host** was configured to use Pi-hole as its DNS server — this created a circular dependency: if Pi-hole's pod ever went down (crash, `ImagePullBackOff`, node restart), the host itself lost DNS resolution and couldn't even pull a new image to fix it. This happened more than once before being addressed.

**Fix**: the host resolves directly against external resolvers (`9.9.9.9`, `1.1.1.1`) via NetworkManager, independent of anything running in the cluster. **LAN clients** (laptops, phones) still use Pi-hole as their DNS server via router DHCP — only the host that runs the cluster was taken out of that loop, since it can't depend on a service it's responsible for keeping alive.

## ✅ Prerequisites

For anyone looking to reproduce this setup:

- Fedora CoreOS installed on the target node(s), provisioned via Ignition.
- Static IP configured for every node.
- k3s installed and running (`kubectl` accessible via `/etc/rancher/k3s/k3s.yaml`).
- `kubectl` CLI installed on the client machine.
- ArgoCD installed in the cluster (`argocd` namespace) and able to reach this GitHub repository.
- `root-app.yaml` and `root-infra.yaml` applied once, manually, to bootstrap the app-of-apps pattern.
- `kubeseal` CLI installed on the client machine (for encrypting secrets before committing them).

## 🚧 Roadmap & Coming Soon

This repository is under active development. Expect upcoming updates including:

- 🔄 **Expanded Services**: Unbound as a recursive upstream resolver for Pi-hole, Home Assistant, n8n, Actual Budget, and eventually an *arr media stack.
- 🧠 **LLM tooling**: wire up llamacpp's OpenAI-compatible API to other services (e.g. n8n workflows).
- 🔧 **Fix ArgoCD UI Ingress**: resolve the HTTPS redirect (see Known Issues above) so `argocd.home` works without port-forwarding.
- 📈 **Monitoring Implementation**: Advanced dashboards and alerting rules with notifications.
- 🔒 **TLS / Ingress**: cert-manager + domain-based routing for services.
- 💾 **Persistent Storage**: StorageClass and PVC strategy for stateful apps.

*Stay tuned for updates!*
