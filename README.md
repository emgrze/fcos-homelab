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
| LAN       | 2.5 Gbps               |

## 🔄 GitOps Workflow

This repository is the **single source of truth** for everything deployed on the cluster. No manual `kubectl apply` for application manifests — only ArgoCD `Application` definitions are applied once, manually, to bootstrap ArgoCD's awareness of a given app. Everything after that is automated:

```
Edit manifests locally
        ↓
git commit & push to GitHub
        ↓
ArgoCD detects the change (auto-sync in ~3 min)
        ↓
ArgoCD applies changes to the cluster
        ↓
Self-heal: any manual drift in the cluster is reverted to match Git
```

Sync policy for all apps: `automated: prune + selfHeal`, meaning:
- Resources removed from Git are pruned from the cluster.
- Any manual changes made directly in the cluster (e.g. `kubectl scale`) are automatically reverted to match the state defined in Git.

## 📁 Repository Structure

```
.
├── apps/
│   └── podinfo/
│       ├── deployment.yaml
│       ├── service.yaml
│       └── kustomization.yaml
└── argocd/
    └── applications/
        └── podinfo-app.yaml
```

- `apps/` — Kubernetes manifests for each application, one folder per app.
- `argocd/applications/` — ArgoCD `Application` CRs, one per app, pointing ArgoCD at the corresponding `apps/<name>` path.

## 🚀 Deployed Apps

| App     | Description                        | Namespace | Status         |
|---------|-------------------------------------|-----------|----------------|
| podinfo | GitOps flow test / demo app         | `apps`    | ✅ Running (LoadBalancer, port 9898) |

## ✅ Prerequisites

For anyone looking to reproduce this setup:

- Fedora CoreOS installed on the target node(s), provisioned via Ignition.
- Static IP configured for every node.
- k3s installed and running (`kubectl` accessible via `/etc/rancher/k3s/k3s.yaml`).
- `kubectl` CLI installed on the client machine.
- ArgoCD installed in the cluster (`argocd` namespace) and able to reach this GitHub repository.

## 🚧 Roadmap & Coming Soon

This repository is under active development. Expect upcoming updates including:

- 🔄 **Expanded Services**: Integration of new self-hosted applications.
- 📈 **Monitoring Implementation**: Advanced dashboards and alerting rules with notifications.
- 🔒 **TLS / Ingress**: cert-manager + domain-based routing for services.
- 💾 **Persistent Storage**: StorageClass and PVC strategy for stateful apps.

*Stay tuned for updates!*
