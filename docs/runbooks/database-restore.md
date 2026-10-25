# Database restore

Restore procedures for the CNPG databases (Nextcloud, Immich). How the dumps are produced: see
[README → Database Backups](../../README.md#database-backups).

**Last restore test:** 2026-10-25 — Nextcloud OK, Immich OK.

---

## Restore test

Run this periodically (and after Immich/Postgres upgrades): restores the latest dump of each
database into a throwaway Postgres pod and compares row counts with production. Production is
only read. Manifests in [`tools/restore-test/`](../../tools/restore-test/) (not managed by ArgoCD).

### Nextcloud

```bash
kubectl apply -f tools/restore-test/nextcloud.yml
kubectl exec -n nextcloud restore-test -- sh -c 'until pg_isready -U postgres; do sleep 1; done; ls -lh /dumps'
kubectl exec -n nextcloud restore-test -- createdb -U postgres nextcloud
kubectl exec -n nextcloud restore-test -- pg_restore -U postgres -d nextcloud --no-owner --no-acl --exit-on-error /dumps/<file>.dump
kubectl exec -n nextcloud restore-test -- psql -U postgres -d nextcloud -c "select count(*) from oc_users;" -c "select count(*) from oc_filecache;"
kubectl delete -f tools/restore-test/nextcloud.yml
```

`--no-owner --no-acl`: the `nextcloud` role does not exist in the throwaway Postgres.

Compare with production:

```bash
kubectl exec -n nextcloud nextcloud-postgres-1 -c postgres -- psql -U postgres -d nextcloud -c "select count(*) from oc_users;" -c "select count(*) from oc_filecache;"
```

### Immich

```bash
kubectl apply -f tools/restore-test/immich.yml
kubectl exec -n immich restore-test -- sh -c 'until pg_isready -U postgres; do sleep 1; done; ls -lh /library/backups | tail -3'
kubectl exec -n immich restore-test -- psql -U postgres -c "create role immich login;" -c "create database immich owner immich;"
kubectl exec -n immich restore-test -- sh -c "gunzip -c /library/backups/<file>.sql.gz | sed -e \"s/SELECT pg_catalog.set_config('search_path', '', false);/SELECT pg_catalog.set_config('search_path', 'public, pg_catalog', true);/g\" -e '/^\\\\restrict /d' -e '/^\\\\unrestrict /d' | psql -U postgres -d immich -v ON_ERROR_STOP=1 > /tmp/restore.log 2>&1; echo exit=\$?; grep -i -E 'error|invalid' /tmp/restore.log | head"
kubectl exec -n immich restore-test -- psql -U postgres -d immich -c "select count(*) from asset;" -c "select count(*) from smart_search;"
kubectl delete -f tools/restore-test/immich.yml
```

Compare with production:

```bash
kubectl exec -n immich immich-postgres-1 -c postgres -- psql -U postgres -d immich -c "select count(*) from asset;" -c "select count(*) from smart_search;"
```

`smart_search` holds the CLIP embeddings: matching counts mean VectorChord data came back too.

### Success criteria

- No errors (`exit=0` for Immich).
- Counts match production. Nextcloud's file count may be slightly lower (files synced after the dump).

---

## Immich dump gotchas

These also apply to a real restore:

- It is a `pg_dump` of the database only — no role, no `CREATE DATABASE`. Create both first
  (in a real restore, the CNPG `bootstrap.initdb` does it).
- It contains `\restrict` / `\unrestrict` (pg_dump ≥ 17.6). Older `psql` (the cluster image ships 17.5)
  stops with `invalid command \restrict` — strip both lines with `sed`.
- Keep the `search_path` `sed` from Immich's docs, otherwise VectorChord functions are not found.

---

## Disaster restore

Restores into the real CNPG cluster. The `restore-test` pod is used only to read the dump from
NFS; the data is streamed into the CNPG pod, where `kubectl exec` runs as the `postgres` superuser
(needed for extensions; no password involved).

> ⚠️ Not yet exercised end to end — the dumps themselves are verified by the restore test above.

1. Stop the app so nothing writes or runs migrations:

   ```bash
   kubectl scale deployment <nextcloud|immich-server> -n <ns> --replicas=0
   ```

2. Make sure the CNPG cluster exists and is empty. A fresh `kubectl apply -f charts/<app>/postgres-cluster.yml`
   creates the role, the database and, for Immich, the extensions.
3. Start the reader pod:

   ```bash
   kubectl apply -f tools/restore-test/<app>.yml
   ```

4. Stream the dump into the cluster:

   ```bash
   # Nextcloud
   kubectl exec -n nextcloud restore-test -- cat /dumps/<file>.dump \
     | kubectl exec -i -n nextcloud nextcloud-postgres-1 -c postgres -- \
       pg_restore -U postgres -d nextcloud --clean --if-exists --exit-on-error

   # Immich
   kubectl exec -n immich restore-test -- cat /library/backups/<file>.sql.gz \
     | gunzip \
     | sed -e "s/SELECT pg_catalog.set_config('search_path', '', false);/SELECT pg_catalog.set_config('search_path', 'public, pg_catalog', true);/g" \
           -e '/^\\restrict /d' -e '/^\\unrestrict /d' \
     | kubectl exec -i -n immich immich-postgres-1 -c postgres -- psql -U postgres -d immich -v ON_ERROR_STOP=1
   ```

   Nextcloud keeps the original owners here (no `--no-owner`): the `nextcloud` role exists in the CNPG cluster.

5. Clean up and bring the app back:

   ```bash
   kubectl delete -f tools/restore-test/<app>.yml
   kubectl scale deployment <nextcloud|immich-server> -n <ns> --replicas=1
   ```
