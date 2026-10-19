# 🏠 Fedora CoreOS Homelab Server

My declarative personal homelab server based on **Fedora CoreOS**, **k3s** and **argocd**. My goal is to have a lightweight, safe and reliable setup for self-hosted services. All fully managed with GitOps (everything in cluster defined/versioned here).

## 🛠 Tech Stack

| Component | Tool / Technology |
| --- | --- |
| OS | Fedora CoreOS (immutable, declarative) |
| Orchestration | k3s (lightweight Kubernetes) |
| gitops / CD | ArgoCD |
| Ingress / Proxy | Traefik (bundled with k3s) |
| LoadBalancing | ServiceLB (bundled with k3s) |
| Image automation | ArgoCD image updater |

## 🖥 Hardware

| Component | Spec |
| --- | --- |
| CPU | AMD Ryzen H255 |
| RAM | 16GB DDR5 |
| iGPU | AMD Radeon 780M |
| LAN | 2.5Gbps |

## 🔄 GitOps workflow

- this repo is the **single source of truth** for everything that gets deployed into the cluster
- there are only two manual `kubectl apply` in entire lifecycle of project (bootstrapping below two root-apps) after that it's fully automated with **app-of-apps**:

```
edit manifests locally
        ↓
git commit & push to Github
        ↓
argocd notices change (auto sync ~3min or webhook?)
        ↓
argocd applies changes in cluster
        ↓
self heal: revert any manual drift in the cluster to match Git.
```

Sync policy for all apps: `automated: prune + selfHeal` so ->

- remove resources from git, it will be pruned from cluster
- Manual changes made directly into the cluster (e.g. kubectl scale) gets automatically reverted back to what is defined in Git.

### Dual app-of-apps - user vs infrastructure apps

two separate root `Application` resource that split responsibility

| Root Application | Watched path | Purpose |
| --- | --- | --- |
| `root` | apps/ | User facing services (podinfo, future Pi-hole, Unbound, HomeAssistant etc.) |
| `root-infra` | argocd/applications/ | Cluster infrastructure components (e.g. sealed secrets, future cert manager, monitoring, ...) |

Both bootstrapped manually once:

```bash
sudo kubectl apply -f root-app.yaml
sudo kubectl apply -f root-infra.yaml
```

From now on when you add new user app or new infrastructure component -> it's just `git push` and no more manual kubectl applies for that component.

## 📁 Repository structure

```
.
├── root-app.yaml                             # bootstraps: watches apps/
├── root-infra.yaml                           # bootstraps: watches argocd/applications/
├── .gitignore                                # exclude raw secrets / kubeseal temp files
├── apps/
│   ├── podinfo/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   └── kustomization.yaml
│   ├── pihole/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── pvc.yaml
│   │   ├── sealed-secret.yaml                # WEBPASSWORD encrypted (safe to commit)
│   │   └── kustomization.yaml
│   ├── llamacpp/
│   │   ├── deployment.yaml                   # llama.cpp server - vulkan, GPU offload using /dev/dri
│   │   ├── service.yaml
│   │   ├── pvc.yaml                          # model storage
│   │   ├── ingress.yaml                      # http://ai.home via Traefik
│   │   └── kustomization.yaml
│   ├── homepage/
│   │   ├── configmap.yaml                    # settings / services / widgets (entire config in git!)
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── ingress.yaml                      # http://panel.home
│   │   ├── kustomization.yaml
│   │   └── .argocd-source-homepage.yaml      # written by Image Updater (git write-back)
│   ├── argocd-ingress/
│   │   ├── ingress.yaml                      # http://argocd.home
│   │   └── kustomization.yaml
│   └── argocd-image-updater/
│       ├── sealed-secret.yaml                # git-creds for write-back (encrypted)
│       ├── imageupdater.yaml                 # ImageUpdater CR (v1.x requires it)
│       └── kustomization.yaml
└── argocd/
    └── applications/
        ├── argocd.yaml                       # Application CR -> argo-cd Helm chart (ArgoCD manages itself)
        ├── argocd-ingress-app.yaml           # Application CR -> apps/argocd-ingress
        ├── argocd-image-updater.yaml         # Application CR -> Image Updater Helm chart
        ├── argocd-image-updater-config.yaml  # Application CR -> apps/argocd-image-updater
        ├── homepage-app.yaml                 # Application CR -> apps/homepage
        ├── llamacpp-app.yaml                 # Application CR -> apps/llamacpp
        ├── pihole-app.yaml                   # Application CR -> apps/pihole
        ├── podinfo-app.yaml                  # Application CR -> apps/podinfo
        └── sealed-secrets.yaml               # application (helm chart) - infra component
```

- `root-app.yaml` / `root-infra.yaml` only two manifests that will ever get applied manually to cluster.
- `apps/` Kubernetes manifests for user facing applications. One folder per app
- `argocd/applications/` ArgoCD `Application` CRs for both user apps and infrastructure components, watched by `root-infra` exclusively.

## 🚀 Deployed Apps

| App | Description | Namespace | Status |
| --- | --- | --- | --- |
| podinfo | GitOps flow test / demo app | apps | ✅ running (LB port:9898) |
| pihole | Network wide ad/tracker blocker - DNS server for LAN | apps | ✅ running (LB -> dns on port 53, web ui on port 8080) |
| llamacpp | local LLM inference (model swappable with one env var), GPU acceleration | apps | ✅ running (ingress: http://ai.home) |
| homepage | Dashboard/landing page for entire homelab | apps | ✅ running (ingress: http://panel.home) |

## 🧩 Infrastructure Components

| Component       | Description                                  | Namespace     | Status    |
|------------------|-----------------------------------------------|----------------|-----------|
| argocd | GitOps controller - manages itself (self-managed `Application`, Helm chart pinned to 10.1.3) | argocd | ✅ running |
| sealed-secrets | Encrypts secrets to be safe to commit to this public repo | kube-system | ✅ running |
| argocd image updater | Automatically update container images digests/tags via git writeback | argocd | ✅ running |

## 🔐 Secrets management

because it's a **public** repository - no way we can ever commit plaintext kubernetes `Secret` objects. We need to encrypt secrets client-side using [sealed secrets](https://github.com/bitnami/sealed-secrets) before any Git interaction happens (only the `sealed-secret-controller` in cluster has private key for decryption).

How to add secret into any app (Pihole, Unbound, HomeAssistant etc.):

1. Create raw secret locally (will never get committed!)

   ```bash
   kubectl create secret generic <app>-secret \
     --namespace=apps \
     --from-literal=KEY='value' \
     --dry-run=client -o yaml > secret-tmp.yaml
   ```

2. Encrypt it with `kubeseal`

   ```bash
   kubeseal \
     --controller-name sealed-secrets-controller \
     --controller-namespace kube-system \
     --scope namespace-wide \
     -f secret-tmp.yaml \
     -w apps/<app>/sealed-secret.yaml
   ```

3. Remove raw file, commit only encrypted one

   ```bash
   rm secret-tmp.yaml
   git add apps/<app>/sealed-secret.yaml
   git commit -m "Add sealed <app> secret"
   git push
   ```

Now you can safely publish `SealedSecret` because it's decryptable by controller on this specific cluster only.

> ⚠️ private key of the controller is backed up **outside** Git (password manager / encrypted storage) -> if lost without backup, then we cannot decrypt any existing `SealedSecret` after rebuilding the cluster!

## 🕳️ Pihole notes

Some deployment stuff that might be useful to note for anyone trying to reproduce it.

- **Persistent storage** - `/etc/pihole`, `/etc/dnsmasq.d` backed by PVC (StorageClass local-path) so config / blocklists / query logs will survive pod restart
- **Port 80 conflict** - since Traefik `svclb` DaemonSet already binds hostPort:80 on this single node cluster, PiHole webui is exposed as **port:8080** (to not collide with that in `LoadBalancer` service scheduling)
- **Pihole v6 env vars:** from v6 image `pihole/pihole` went to CalVer versioning which renamed most of the env vars. e.g. WEBPASSWORD -> FTLCONF_webserver_api_password. This deployment uses new names
- **Upstream DNS** - configured with quad9 9.9.9.9 as upstream resolver (will be replaced with local Unbound for full recursive resolution once it's deployed)
- **Password:** set up using `SealedSecret` following the Secrets management explained above

## 🤖 Local LLM inference notes

Running [llamacpp](https://github.com/ggml-org/llama.cpp) - GPU acceleration on AMD Radeon 780M iGPU **Vulkan** backend (vs ROCm). Reason: Vulkan for this gpu is mature and no unofficial hacks needed (radv / mesa), while ROCm support for gfx1103 is still experimental stuff. Some things to note here, that might be useful for anyone trying to reproduce it.

- **GPU passthrough** - `/dev/dri` passed to container using `hostPath`. Container set as `privileged: true`. It's the simplest working setup (good future idea would be scope down to something like supplementalGroups and use host render group GID instead of full privileged).
- **Known bug in official image.** `ghcr.io/ggml-org/llama.cpp:server-vulkan` can't find any vulkan devices [issue](https://github.com/ggml-org/llama.cpp/issues/24651) (but exact same Vulkan binary works fine when run manually inside the container). So this deployment uses community maintained image `ghcr.io/kth8/llama-server-vulkan` which doesn't have that problem.
- **Shared memory budget** - 780m is UMA no dedicated vram, so whatever GPU memory we get from BIOS allocating it out of system RAM (only a few GB free after driver overhead here). So for example Qwen3.5-9B (~5.6GB) didn't fit and silently fell back to partial cpu inference (~7 tok/s). **Gemma 4 E2B** (~3.1GB) fitted nicely (~23tok/s). If we have model around the budget (e.g ~5.3GB), then it still works on GPU - no CPU load during inference, but with reduced throughput (~13 tok/s). So there is probably a soft limit somewhere between 3-5gb where gpu offload starts to be squeezed.
- **Single source of truth for model:** initContainer fetches the model from env var `MODEL_URL` (not hardcoded in image) so it's easy to swap models -> just one line change + git push and no more manual kubectl applies needed for that component. Also chat ui auto derives name of model based on url filename, including quants - writes into shared file which main container reads at start. No need for separate manually updated alias etc...
- **Atomic/fail fast downloads:** we do the download to .tmp file only `mv` in place if it succeeds (so no corrupted / partial model after interrupted restart). In an earlier version of this without `set -e` in init script, we were masking failed curl e.g. because of transient DNS issue as "Download complete" false positive and then got a missing model silently... 700+ CrashLoopBackOff restarts until the reason was figured out :) Now it aborts immediately on any command failure in init script.
- **Monitoring progress of init container with big files download:** kubectl logs -f for init container can look like stuck when large file is being downloaded (curl updates its progress bar using \r carriage return, sometimes doesn't stream properly through k8 log api) So we can check more reliably by doing `kubectl exec <pod> -c <initcontainer> -- stat -c%s /path/to/file.tmp` twice after a few seconds if it actually makes progress with the download.
- **Access:** exposed via Traefik Ingress as http://ai.home (added local dns record in pihole) instead of LoadBalancer port to avoid another dedicated ip+port and not repeat Port 80 conflict issue like we had for Pi-hole webui. Note: first tried .local, it didn't work - that's reserved mDNS domain which most systems intercept before normal DNS resolver even gets a chance at it...
- **API OpenAI compatible** - on the same port as built in web chat ui `/v1/chat/completions` etc. useful to wire up future services like n8n

## 📊 Homepage Dashboard notes

Decided for [homepage](https://gethomepage.dev) over e.g. Homarr because their entire config is plain yaml files (settings.yaml, services.yaml, widgets.yaml), not some state inside the container database - so it's consistent with gitops we have everywhere in this repo: config stored in ConfigMap which is mounted into pod and fully versioned.

- **Changing ConfigMap requires manual restart of pod** - if you update `ConfigMap` in Git and let ArgoCD sync -> then file on disk (inside the pod) will be updated, with a short kubelet propagation delay - but homepage doesn't notice changes and automatically reload it. After any change to configmap.yaml: git push → wait for argocd sync → `kubectl rollout restart deployment <homepage> -n apps` → click revalidate icon in ui (or hard-refresh browser) because home page is statically rendered / cached (NextJS SSG). All 3 of these steps are important! If you skip any one, then it looks like change 'didn't do anything'.
- **`style: row` forces group to be full width of the page** - so we can't have multiple groups side by side.

  If you don't set `style:row` (just `columns:N`) then homepage with its flex based layout will try to put groups next to each other if it has space. Not obvious in docs and looks like caching issue when wrong...

- **Pihole v6 widget syntax** - needs explicit field `version: 6` + pass API key/pass via `key`, because pihole v6 replaced old token api `/admin/api.php`. So we need to have the *port* in widget's url (e.g. :8080) if it differs from service default port - not obvious in docs, initially caused silent timeouts of connection...
- **Reuse existing secret:** instead of creating another SealedSecret with duplicate password for Pihole widget -> using same `pihole-secret` / WEBPASSWORD key as we use for actual pihole deployment. It's safe because that one is sealed with --scope namespace wide and it means in future when we want to rotate the password - only need to do it once :)

## 🔁 ArgoCD Image Updater notes

Added this to avoid manual bumping image tags (and get rid of :latest without losing ability to actually get new versions). Some non obvious things from getting it working...

- **v1.x has CRD, not just annotations**: Older tutorials / blogposts setup Image updater purely with `argocd-image-updater.argoproj.io/*` annotations on `Application`. Now in v1.0.0+ the controller reconciles dedicated custom resource ImageUpdater
  If you don't use it (CR) then the controller logs "No ImageUpdater CRs to process" and does nothing silently... If Application has an annotation, only if that ImageUpdater CR explicitly opted-in for using annotations with `useAnnotations:true` on matching `applicationRef`.
- **digest update strategy requires explicit tag in image-list.** e.g. image=repo/image (no tag) will fail "cannot use 'digest' as update strategy ... without version constraint" -> needs to be image=repo/image:latest
- **For Git writeback you need labeled secret**: Credentials `Secret` must have label `argocd.argoproj.io/secret-type=repo-creds` or ImageUpdater won't recognize it, even though writeback annotation correctly points e.g. `git:secret:<namespace>/<name>`.
- **Writeback for Kustomize apps** will be in auto generated file `.argocd-source-<app>.yaml` next to app manifests (not manually edited kustomization.yaml). This file gets committed by bot and should *not* be excluded with .gitignore.
- **Expect diverged branch:** Bot commits directly to main whenever finds new digest, independently of local work... Can have git push fail "fetch first" if bot did push in between.. use `git pull` (with either merge or rebase for pull.rebase since newer git requires explicit choice). It should be safe as long as you didn't touch same files on both sides.

## ⚠️ Known issue: ArgoCD ui Ingress redirects to https -> resolved

Initially we had Ingress for argocd-ui http://argocd.home and it would redirect with 307 to HTTPS (no valid cert behind) since argocd-server is run with TLS enabled by default... Fixed by patching `server.insecure: "true"` into `argocd-cmd-params-cm` and restarting argocd-server

> ✅ ArgoCD now manages itself - `argocd` Application (`argocd/applications/argocd.yaml`) points at the `argo-cd` Helm chart (pinned to `10.1.3`) with the same release name as the original `helm install`, so it simply adopted the running resources. `server.insecure` lives in the Application's `values` now, so it's in Git and survives chart upgrades. `ServerSideApply=true` is required, because ApplicationSet CRD is too big for client-side apply. The old Helm release record was deleted after adoption - don't `helm upgrade` / `helm uninstall` this anymore (uninstall would remove the actual resources). To upgrade ArgoCD: bump `targetRevision` in Git (check the chart changelog first). If the controller ever breaks and can't sync itself: `helm template argocd argo/argo-cd --version 10.1.3 -n argocd -f values.yaml | kubectl apply --server-side --force-conflicts -f -`. (The Helm logo next to the app in UI is just the source type, not Helm managing it.)

## 🌐 DNS architecture note: host is *not* using pihole as resolver

Initially on the k3s **host** we had it configured to use Pi-hole for DNS. Then there was this circular dependency, because if somehow pod of Pi-hole would go down (e.g. crashed / ImagePullBackOff / node restart) then host itself couldn't do DNS resolution anymore and can't even pull new image to fix that! And we had more than one instance of that before it got resolved...

**Fix:** Now the host is resolving directly against external resolver 9.9.9.9, 1.1.1.1 etc. using NetworkManager independent of anything in cluster. **LAN clients** (laptops / phones) still use pihole as DNS server via router DHCP - just removed host which runs cluster from that chain because it can't rely on something we're responsible to keep running...

## ✅ Prerequisites

For anyone trying to reproduce this:

- Fedora CoreOS installed onto target node(s). Provisioned using Ignition.
- Static IP configured for each node
- k3s installed + running (kubectl accessible at /etc/rancher/k3s/k3s.yaml)
- kubectl CLI on client machine
- ArgoCD in cluster (namespace argocd) - can access this Github repo
- root-app.yaml and root-infra.yaml applied once manually to bootstrap app of apps :)
- Kubeseal cli for client machine, to encrypt secrets before we commit them...

## 🚧 Roadmap & Coming Soon

This is ongoing repository so expect updates with:

- 🔄 **Expanded services:** Unbound as recursive upstream resolver for pihole Homeassistant Actual Budget Eventually some arr media stack.
- 🧠 **Agentic AI layer** - Once we increase the budget of shared memory on iGPU and have larger model which can do agent stuff, wire llamacpp OpenAi compatible API into something like agent orchestration (e.g. **Hermes** or **OpenClaw**) instead of just general workflow tool like n8n
- 📦 **NAS / Private cloud** - Self-hosted file storage/sync service. Probably with dedicated storage rather than local-path etc...
- 🌍 **Remote access:** secure way to access homelab from outside LAN - looking at options (e.g. **Tailscale** vs **Netbird**)
- 📈 **Monitoring implementation:** Advanced dashboards, alerting rules.. notifications!
- 🔒 **TLS / Ingress:** cert manager + domain based routing for services
- 💾 **Persistent storage:** StorageClass/PVC strategy for stateful apps

*Stay tuned for updates!*
