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
| CloudNativePG | PostgreSQL operator (used by Immich) |

---

## Services

| Service | URL | Notes |
|---|---|---|
| Nextcloud | `nextcloud.homelab.luisbrancher.dev` | File sync, source of truth for files |
| Immich | `immich.homelab.luisbrancher.dev` | Photo library (v3.0.0), CNPG-backed Postgres with VectorChord, reads Nextcloud photos as External Library (pending) |
| Jellyfin | `jellyfin.homelab.luisbrancher.dev` | Media server, download/watch/delete workflow |
| cert-manager | — | Cloudflare DNS-01 challenge, Let's Encrypt (v1.20.x) |
| cnpg-operator | — | Manages the PostgreSQL `Cluster` used by Immich |

---

## Repository Structure

```
homelab-gitops/
├── bootstrap/
│   ├── argocd-app-of-apps.yml    # Single entry point — applied once manually
│   └── namespaces.yml            # Cluster namespaces — applied once manually
│
├── apps/                         # ArgoCD watches this directory
│   ├── cert-manager.yml
│   ├── cnpg-operator.yml
│   ├── nextcloud.yml
│   ├── immich.yml
│   └── jellyfin.yml
│
├── charts/                       # Helm values and PVCs per service
│   ├── nextcloud/
│   │   ├── values.yml
│   │   └── pvc.yml
│   ├── immich/
│   │   ├── values.yml
│   │   ├── pvc.yml
│   │   └── postgres-cluster.yml  # CNPG Cluster CRD (Postgres + VectorChord)
│   └── jellyfin/
│       ├── values.yml
│       └── pvc.yml
│
└── config/
    └── cert-manager/
        ├── cluster-issuer.yml
        └── cloudflare-secret.yml  # Placeholder only — real value applied via CLI
```

---

## How It Works

ArgoCD monitors the `apps/` directory. Any `Application` manifest placed there is automatically picked up and synced. Adding a new service means creating `apps/new-service.yml` and pushing to `main`.

```
git push → ArgoCD detects change → Helm sync → cluster updated
```

The `bootstrap/` files are the only resources applied manually — everything else is GitOps.

Application manifests that reference Helm values from this repo use the ArgoCD multi-source pattern (`sources:` + `ref: values`), since chart and values live in different repositories.

---

## Initial Deploy Order

```bash
# 1. Create namespaces
kubectl apply -f bootstrap/namespaces.yml

# 2. Bootstrap ArgoCD App of Apps
kubectl apply -f bootstrap/argocd-app-of-apps.yml

# 3. Wait for cnpg-operator to be ready (ArgoCD syncs it from apps/)
kubectl get pods -n cnpg-system

# 4. Apply PVCs (NFS + hostPath volumes)
kubectl apply -f charts/nextcloud/pvc.yml
kubectl apply -f charts/immich/pvc.yml
kubectl apply -f charts/jellyfin/pvc.yml

# 5. Create the Immich Postgres credentials secret (before the Cluster)
kubectl create secret generic immich-postgres-credentials \
  --namespace immich \
  --from-literal=username=immich \
  --from-literal=password=<senha-forte-aqui>

# 6. Apply the CNPG Cluster for Immich
kubectl apply -f charts/immich/postgres-cluster.yml

# 7. Apply cert-manager config (after cert-manager pods are healthy)
kubectl apply -f config/cert-manager/cluster-issuer.yml

# 8. Create remaining secrets (never committed to git)
kubectl create secret generic nextcloud-credentials \
  --namespace nextcloud \
  --from-literal=username=<user> \
  --from-literal=password=<password>

kubectl create secret generic nextcloud-db-credentials \
  --namespace nextcloud \
  --from-literal=password=<senha-do-banco>

kubectl create secret generic immich-credentials \
  --namespace immich \
  --from-literal=secret=<jwt-secret>

kubectl create secret generic cloudflare-api-token \
  --namespace cert-manager \
  --from-literal=api-token=<token>
```

> **TODO:** `immich-nextcloud-photos-pvc` está criado mas ainda não montado no Immich.
> Para ativar como External Library, adicionar `extraVolumes` e `extraVolumeMounts` no
> `charts/immich/values.yml` e configurar o path na UI do Immich após o primeiro deploy.

---

## Secrets

Secrets are **never stored in this repository**. They are applied directly via `kubectl create secret` on first deploy.

Future improvement: encrypt secrets with [SOPS](https://github.com/getsops/sops) and commit safely.

---

## Storage

Persistent data lives on a dedicated NAS (TrueNAS, NFS, SSD pool `data-ssd`). Jellyfin media uses local NVMe storage (download, watch, delete workflow). Immich's Postgres database runs on local storage (not NFS — Postgres on NFS is not recommended due to fsync/locking behavior); the photo/video library itself stays on the NFS share.

| PVC | Backend | Size |
|---|---|---|
| `nextcloud-pvc` | NFS `10.10.10.15:/mnt/data-ssd/nextcloud` | 200Gi |
| `immich-pvc` | NFS `10.10.10.15:/mnt/data-ssd/immich` | 100Gi |
| `immich-nextcloud-photos-pvc` | NFS `10.10.10.15:/mnt/data-ssd/nextcloud/photos` (read-only) | 200Gi |
| `jellyfin-config-pvc` | NFS `10.10.10.15:/mnt/data-ssd/jellyfin` | 10Gi |
| `jellyfin-media-pvc` | hostPath `/mnt/media` (NVMe) | 200Gi |
| Immich Postgres (CNPG-managed) | local-path (k3s default StorageClass) | 20Gi |

---

## Related

- [homelab-infra](https://github.com/luisbrancher/homelab-infra) — Terraform + Ansible
- [luisbrancher.dev](https://luisbrancher.dev) — personal site and blog