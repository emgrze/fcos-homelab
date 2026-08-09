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
│   └── llamacpp/
│       ├── deployment.yaml       # llama.cpp server-vulkan, GPU offload via /dev/dri
│       ├── service.yaml
│       ├── pvc.yaml              # model storage
│       ├── ingress.yaml          # exposed at http://ai.home via Traefik
│       └── kustomization.yaml
└── argocd/
    └── applications/
        ├── podinfo-app.yaml     # Application CR pointing at apps/podinfo
        ├── pihole-app.yaml      # Application CR pointing at apps/pihole
        ├── llamacpp-app.yaml    # Application CR pointing at apps/llamacpp
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
| llamacpp | Local LLM inference (Gemma 4 E2B) with GPU acceleration | `apps` | ✅ Running (Ingress — `http://ai.home`) |

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
- **Shared memory budget**: the 780M uses UMA (no dedicated VRAM) — available GPU memory is whatever the BIOS allocates from system RAM, in this case only a few GB free after driver overhead. A 9B model (Qwen3.5-9B, ~5.6GB) didn't fully fit and silently fell back to partial CPU inference (~7 tok/s). Switching to **Gemma 4 E2B** (~3.1GB, Q4_K_M) fit comfortably in GPU memory and brought throughput up to **~23 tok/s**.
- **Access**: exposed via Traefik `Ingress` at `http://ai.home` (a local DNS record added in Pi-hole), rather than a `LoadBalancer` port, to avoid consuming another dedicated IP/port and to avoid a repeat of the port 80 conflict seen with Pi-hole's web UI. Note: `.local` was tried first and didn't work — it's a reserved mDNS domain that most systems intercept before it ever reaches a normal DNS resolver.
- **API**: OpenAI-compatible, served on the same port as the built-in web chat UI (`/v1/chat/completions`, etc.) — useful for wiring up future services like n8n.

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
- 📈 **Monitoring Implementation**: Advanced dashboards and alerting rules with notifications.
- 🔒 **TLS / Ingress**: cert-manager + domain-based routing for services.
- 💾 **Persistent Storage**: StorageClass and PVC strategy for stateful apps.

*Stay tuned for updates!*
