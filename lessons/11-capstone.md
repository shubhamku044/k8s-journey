# Lesson 11 — Capstone: Multi-Tier App ✅ 🎉

Built and debugged a complete two-tier app on Kubernetes, entirely declarative, in the `capstone`
namespace. No new concepts — pure synthesis of Lessons 1–10.

## Architecture
```
   [ Adminer ]  Deployment + NodePort Service  (stateless app tier — exposed to browser)
       │  reads ADMINER_DEFAULT_SERVER=db from a ConfigMap → finds the DB by Service DNS
       ▼
   [ PostgreSQL ]  Deployment + ClusterIP Service 'db'  (data tier — internal only)
       │  POSTGRES_PASSWORD from a Secret
       ▼
   [ PersistentVolumeClaim ]  data survives Pod restarts
```

## How every lesson shows up
- **Deployment / ReplicaSet** (L3) — both tiers.
- **Service** (L4) — ClusterIP `db` (internal) + NodePort `adminer` (external); tiers connect via
  the `db` DNS name (service discovery).
- **ConfigMap** (L5) — `ADMINER_DEFAULT_SERVER=db` (non-sensitive, in git).
- **Secret** (L5) — `POSTGRES_PASSWORD` injected via `secretKeyRef` (created imperatively, NOT in git).
- **PVC / PV / StorageClass** (L6) — durable Postgres storage.
- **Namespace** (L7) — everything isolated in `capstone`.
- **Probes** (L9) — readiness gates traffic; liveness keeps it alive.
- **Requests/limits** (L10) — set on the app tier.

Manifests: [db.yaml](../manifests/11-capstone/db.yaml) · [app.yaml](../manifests/11-capstone/app.yaml)

## Two real bugs I hit and fixed (the best learning)
1. **`exec` probe using `curl`** → the `adminer` image has no `curl`, so the probe always failed.
   **Fix:** use `httpGet` — the kubelet performs the HTTP check itself, no binary needed in the image.
2. **Probe timeouts crash-looped a healthy app** → httpGet's default `timeoutSeconds` is **1s**, but
   adminer's single-threaded PHP dev server sometimes responds slower, so the **liveness** probe
   failed and kept restarting a perfectly fine Pod.
   **Fix:** `timeoutSeconds: 5` on readiness, and a lightweight **`tcpSocket`** liveness probe
   (just check the port is open). Pattern: *lightweight liveness, meaningful readiness.*

## Interview-ready takeaways
- I can whiteboard a multi-tier deployment and how the objects reference each other (Service DNS
  for tier-to-tier, Secret/ConfigMap for config, PVC for state).
- Diagnose a stuck Pod via `kubectl describe` Events + `kubectl logs` (found "context deadline
  exceeded" = probe timeout, not a wrong port).
- Probe design: exec needs the binary in the image; httpGet/tcpSocket run from the kubelet; an
  over-aggressive liveness probe can crash-loop a healthy app.
