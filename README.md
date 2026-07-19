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
| CPU       | AMD Ryzen H255 (chiński/OEM odpowiednik Ryzen 7 8745HS) |
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
├── apps/
│   └── podinfo/
│       ├── deployment.yaml
│       ├── service.yaml
│       └── kustomization.yaml
└── argocd/
    └── applications/
        ├── podinfo-app.yaml     # Application CR pointing at apps/podinfo
        └── sealed-secrets.yaml  # Application CR (Helm chart, infra component)
```

- `root-app.yaml` / `root-infra.yaml` — the only two manifests ever applied manually to the cluster.
- `apps/` — Kubernetes manifests for user-facing applications, one folder per app.
- `argocd/applications/` — ArgoCD `Application` CRs for both user apps and infrastructure components, watched exclusively by `root-infra`.

## 🚀 Deployed Apps

| App     | Description                        | Namespace | Status         |
|---------|-------------------------------------|-----------|----------------|
| podinfo | GitOps flow test / demo app         | `apps`    | ✅ Running (LoadBalancer, port 9898) |

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

- 🔄 **Expanded Services**: Integration of new self-hosted applications (Pi-hole + Unbound, Home Assistant).
- 📈 **Monitoring Implementation**: Advanced dashboards and alerting rules with notifications.
- 🔒 **TLS / Ingress**: cert-manager + domain-based routing for services.
- 💾 **Persistent Storage**: StorageClass and PVC strategy for stateful apps.

*Stay tuned for updates!*
