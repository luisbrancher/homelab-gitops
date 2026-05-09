# homelab-gitops

GitOps repository for my homelab Kubernetes cluster. Managed by ArgoCD using the App of Apps pattern — every service is declared here and synced automatically on `git push`.

For infrastructure provisioning (Terraform + Ansible), see [homelab-infra](https://github.com/luisbrancher/homelab-infra).

---

## Stack

| Tool | Role |
|---|---|
| k3s | Lightweight Kubernetes |
| ArgoCD | GitOps continuous delivery |
| Helm | Package management |
| cert-manager | Automatic TLS via Let's Encrypt |
| Traefik | Ingress controller (bundled with k3s) |

---

## Services

| Service | URL | Notes |
|---|---|---|
| Nextcloud | `nextcloud.homelab.luisbrancher.dev` | File sync, source of truth for files |
| Immich | `immich.homelab.luisbrancher.dev` | Photo library, reads Nextcloud photos as External Library |
| Jellyfin | `jellyfin.homelab.luisbrancher.dev` | Media server, download/watch/delete workflow |
| cert-manager | — | Cloudflare DNS-01 challenge, Let's Encrypt |

---

## Repository Structure

```
homelab-gitops/
├── bootstrap/
│   ├── argocd-app-of-apps.yaml   # Single entry point — applied once manually
│   └── namespaces.yaml           # Cluster namespaces — applied once manually
│
├── apps/                         # ArgoCD watches this directory
│   ├── cert-manager.yaml
│   ├── nextcloud.yaml
│   ├── immich.yaml
│   └── jellyfin.yaml
│
├── charts/                       # Helm values and PVCs per service
│   ├── nextcloud/
│   │   ├── values.yaml
│   │   └── pvc.yaml
│   ├── immich/
│   │   ├── values.yaml
│   │   └── pvc.yaml
│   └── jellyfin/
│       ├── values.yaml
│       └── pvc.yaml
│
└── config/
    └── cert-manager/
        ├── cluster-issuer.yaml
        └── cloudflare-secret.yaml  # Placeholder only — real value applied via CLI
```

---

## How It Works

ArgoCD monitors the `apps/` directory. Any `Application` manifest placed there is automatically picked up and synced. Adding a new service means creating `apps/new-service.yaml` and pushing to `main`.

```
git push → ArgoCD detects change → Helm sync → cluster updated
```

The `bootstrap/` files are the only resources applied manually — everything else is GitOps.

---

## Initial Deploy Order

```bash
# 1. Create namespaces
kubectl apply -f bootstrap/namespaces.yaml

# 2. Bootstrap ArgoCD App of Apps
kubectl apply -f bootstrap/argocd-app-of-apps.yaml

# 3. Apply PVCs (NFS + hostPath volumes)
kubectl apply -f charts/nextcloud/pvc.yaml
kubectl apply -f charts/immich/pvc.yaml
kubectl apply -f charts/jellyfin/pvc.yaml

# 4. Apply cert-manager config (after cert-manager pods are healthy)
kubectl apply -f config/cert-manager/cluster-issuer.yaml

# 5. Create secrets (never committed to git)
kubectl create secret generic nextcloud-credentials \
  --namespace nextcloud \
  --from-literal=username=<user> \
  --from-literal=password=<password>

kubectl create secret generic immich-credentials \
  --namespace immich \
  --from-literal=secret=<jwt-secret>

kubectl create secret generic cloudflare-api-token \
  --namespace cert-manager \
  --from-literal=api-token=<token>
```

---

## Secrets

Secrets are **never stored in this repository**. They are applied directly via `kubectl create secret` on first deploy.

Future improvement: encrypt secrets with [SOPS](https://github.com/getsops/sops) and commit safely.

---

## Storage

Persistent data lives on a dedicated NFS server (SATA 1TB). Jellyfin media uses local NVMe storage (download, watch, delete workflow).

| PVC | Backend | Size |
|---|---|---|
| `nextcloud-pvc` | NFS `/mnt/data/nextcloud` | 200Gi |
| `immich-pvc` | NFS `/mnt/data/immich` | 100Gi |
| `immich-nextcloud-photos-pvc` | NFS `/mnt/data/nextcloud/photos` (read-only) | 200Gi |
| `jellyfin-config-pvc` | NFS `/mnt/data/jellyfin` | 10Gi |
| `jellyfin-media-pvc` | hostPath `/mnt/media` (NVMe) | 200Gi |

---

## Related

- [homelab-infra](https://github.com/luisbrancher/homelab-infra) — Terraform + Ansible
- [luisbrancher.dev](https://luisbrancher.dev) — personal site and blog
