# nas-pi-cluster

A single-node K3s homelab on a Raspberry Pi, managed via GitOps with ArgoCD. Apps are deployed automatically from Git, secrets are encrypted in-repo with Sealed Secrets, and dependency updates flow through Renovate with Telegram notifications.

## What's running

| Component | Purpose |
|-----------|---------|
| **K3s** | Lightweight Kubernetes distribution |
| **ArgoCD** | GitOps continuous deployment |
| **Traefik** | Ingress controller (built into K3s) |
| **cert-manager** | TLS certificates via Let's Encrypt + Cloudflare DNS-01 |
| **Sealed Secrets** | Encrypted secrets safe to commit to Git |
| **Prometheus + Grafana** | Monitoring and dashboards |
| **Tailscale subnet router** | VPN access to cluster services |
| **Plex** | Media server |
| **Recepten** | Custom recipe app |

## Architecture

App-of-apps GitOps pattern:

```
deployments/
├── application.yaml          # Root Application — discovers all sub-apps
├── argocd/argocd/            # Each sub-app is a local Helm chart
├── cert-manager/cert-manager/
├── plex/plex/
│   ├── application.yaml      # ArgoCD Application
│   ├── Chart.yaml            # Chart metadata
│   └── templates/            # Raw manifests, rendered by Helm
└── ...
```

The root Application watches `*/*/application.yaml` files and deploys each as its own ArgoCD Application. Each app is structured as a minimal local Helm chart so manifests live in `templates/` as separate files (per-resource).

Two flavors of charts are used:
- **External charts**: `Chart.yaml` declares a `dependencies:` entry (e.g. argo-cd, grafana, prometheus, cert-manager, sealed-secrets). Versions are pinned for controlled upgrades.
- **Self-contained charts**: own `templates/` directory with raw Kubernetes manifests (e.g. plex, recepten, tailscale subnet router).

## Prerequisites

- Raspberry Pi (4 or newer recommended) with a 64-bit OS and an attached storage volume
- Cloudflare-managed domain (for DNS-01 TLS)
- Tailscale account (for VPN)
- GitHub account (for hosting + Actions)
- Local machine with `ansible`, `kubectl`, `kubeseal`, `helm`

## Initial setup

### 1. Provision the Pi

Edit [inventory.yml](inventory.yml) with your Pi's IP and SSH user, and [host_vars/nas-pi/vars.yml](host_vars/nas-pi/vars.yml) with your domain, mount path, and base hostname. Then run:

```bash
ansible-playbook site.yml
```

This installs K3s with btrfs subvolume for `/var/lib/rancher/k3s`, configures kubeconfig for the user, and installs Tailscale.

### 2. Copy kubeconfig locally

```bash
scp <user>@<pi-ip>:~/.kube/config ~/.kube/config
```

### 3. Bootstrap ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 4. Bootstrap Sealed Secrets controller

Install via Helm so you can decrypt repo-committed sealed secrets (you'll still need to re-encrypt secrets with **your own** cluster's key — see below):

```bash
helm install sealed-secrets oci://registry-1.docker.io/bitnamicharts/sealed-secrets -n sealed-secrets --create-namespace
```

### 5. Apply the root Application

```bash
kubectl apply -f deployments/application.yaml
```

ArgoCD discovers all sub-apps and deploys them.

### 6. Re-encrypt secrets

The SealedSecrets in this repo are encrypted with **my** cluster's private key. You can't decrypt them — they will fail to apply on your cluster. Replace each with secrets encrypted by **your** controller:

```bash
echo -n "my-secret-value" | kubectl create secret generic my-secret \
  --from-file=key=/dev/stdin --dry-run=client -o yaml \
  | kubeseal -o yaml > my-sealed-secret.yaml
```

Apps that need this:
- `deployments/cert-manager/cert-manager-config/templates/cert-manager-sealed-secret.yaml` — Cloudflare API token
- `deployments/observability/grafana/grafana-sealed-secret.yaml` — Grafana admin password
- `deployments/tailscale/subnet-router/templates/tailscale-sealed-secret.yaml` — Tailscale auth key

## Adding a new app

```
deployments/<group>/<appname>/
├── application.yaml       # ArgoCD Application (Helm mode)
├── Chart.yaml             # Chart metadata
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    └── ingress.yaml
```

`application.yaml` skeleton:

```yaml
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: git@github.com:<you>/nas-pi-cluster.git
    path: deployments/<group>/<appname>
    targetRevision: HEAD
    helm:
      releaseName: myapp
  destination:
    server: "https://kubernetes.default.svc"
    namespace: myapp
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Commit + push — the root Application picks it up on the next refresh.

## Update flow (Renovate + Telegram)

Renovate scans weekly (Monday 05:00 UTC) via GitHub Actions and detects new versions for:
- Helm chart dependencies pinned in `Chart.yaml`
- Container image tags in `templates/*.yaml` (skips `:latest`)
- GitHub Actions in `.github/workflows/*.yml`

Available updates appear in the **Dependency Dashboard** issue. Tick a checkbox to create a PR immediately, or wait for the schedule. Merge the PR → ArgoCD deploys.

Telegram notifications fire on:
- 🔀 PR opened / closed / merged
- 📋 Issue opened (e.g. Dependency Dashboard creation)
- 🤖 Renovate scan completion (heartbeat)
- 🔥 Workflow failure

### Required GitHub secrets

| Secret | Source |
|--------|--------|
| `TELEGRAM_BOT_TOKEN` | [@BotFather](https://t.me/botfather) → `/newbot` |
| `TELEGRAM_CHAT_ID` | [@userinfobot](https://t.me/userinfobot) → `/start` |
| `RENOVATE_TOKEN` | [Fine-grained PAT](https://github.com/settings/personal-access-tokens/new) with `contents`, `issues`, `pull-requests`, `workflows` (read+write) |

Add via **Settings → Secrets and variables → Actions**.

## Repository layout

```
.
├── ansible.cfg
├── inventory.yml             # Pi connection details
├── host_vars/nas-pi/vars.yml # Per-host variables (domain, paths)
├── site.yml                  # Ansible playbook entry point
├── roles/
│   ├── k3s/                  # K3s installation
│   └── tailscale/            # Tailscale installation
├── deployments/              # All ArgoCD-managed apps
│   └── application.yaml      # Root app-of-apps
├── renovate.json             # Renovate config
└── .github/workflows/
    ├── renovate.yml          # Weekly Renovate scan
    ├── telegram-notify.yml   # PR/issue events → Telegram
    └── workflow-status.yml   # Workflow completion → Telegram
```

## Useful commands

```bash
# ArgoCD app status
kubectl get applications -n argocd

# Force a hard refresh on an app (busts repo-server cache)
kubectl annotate application <app> -n argocd argocd.argoproj.io/refresh=hard --overwrite

# Watch deploy progress
kubectl get pods -A -w

# Encrypt a secret for committing
kubeseal -f my-secret.yaml -o yaml > my-secret-sealed.yaml
```

## License

Personal homelab — feel free to fork and adapt.
