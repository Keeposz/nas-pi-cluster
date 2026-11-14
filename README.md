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
| **Renovate** | Daily dependency updates via GitHub Actions, patch and digest bumps merge automatically |
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
├── media/plex/
│   ├── application.yaml      # ArgoCD Application
│   ├── Chart.yaml            # Chart metadata (declares upstream plex-media-server as dependency)
│   └── ...
└── ...
```

The root Application watches `*/*/application.yaml` files and deploys each as its own ArgoCD Application. Each app is structured as a minimal local Helm chart so manifests live in `templates/` as separate files (per-resource).

Two flavors of charts are used:
- **External charts**: `Chart.yaml` declares a `dependencies:` entry (e.g. argo-cd, grafana, prometheus, cert-manager, sealed-secrets). Versions are pinned for controlled upgrades.
- **Self-contained charts**: own `templates/` directory with raw Kubernetes manifests (e.g. recepten, tailscale subnet router, *arr stack).

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

Renovate runs in **GitHub Actions** via [.github/workflows/renovate.yml](.github/workflows/renovate.yml): every day at 04:00 UTC, or on demand with **Run workflow** in the Actions tab. It scans `Keeposz/nas-pi-cluster` for:
- Helm chart dependencies pinned in `Chart.yaml`
- Container image tags in `deployments/**/templates/*.yaml` (skips `:latest`)
- GitHub Actions in `.github/workflows/*.yml`

Config lives in [renovate.json](renovate.json). Patch and digest updates merge automatically; minor and major updates open a PR for review. Renovate merges at most two updates per run, the rest follow in the next runs. Merge a PR → ArgoCD deploys.

Each run writes a job summary with what was merged and which updates are still open. Open updates also appear in the **Dependency Dashboard** issue, where ticking a checkbox creates the PR on the next run.

Telegram notifications fire on:
- 🔀 PR opened / closed / merged
- 📋 Issue opened (e.g. Dependency Dashboard creation)
- 🔥 GitHub workflow failure

### Required GitHub secrets

| Secret | Source |
|--------|--------|
| `TELEGRAM_BOT_TOKEN` | [@BotFather](https://t.me/botfather) → `/newbot` |
| `TELEGRAM_CHAT_ID` | [@userinfobot](https://t.me/userinfobot) → `/start` |
| `RENOVATE_TOKEN` | Fine-grained PAT, see below |

Add via **Settings → Secrets and variables → Actions**.

### Renovate GitHub PAT

Renovate needs a Fine-grained PAT to clone the repo and open and merge PRs. Create one at [github.com/settings/personal-access-tokens/new](https://github.com/settings/personal-access-tokens/new) scoped to `nas-pi-cluster` with these repository permissions (read+write where applicable): `Contents`, `Pull requests`, `Issues`, `Workflows`, `Metadata` (read-only). Then store it as an Actions secret (`gh` prompts for the value, so it stays out of your shell history):

```bash
gh secret set RENOVATE_TOKEN --repo Keeposz/nas-pi-cluster
```

The PAT expires; when Renovate runs start failing with 401, create a new one and set it again.

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
    ├── renovate.yml          # Daily Renovate run + job summary
    ├── telegram-notify.yml   # PR/issue events → Telegram
    └── workflow-status.yml   # Workflow failure → Telegram
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

Personal homelab --> feel free to fork and adapt.
