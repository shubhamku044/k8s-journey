# Lesson 05 — ConfigMaps & Secrets ✅

## The problem
Config (DB URLs, feature flags, API keys) must NOT be baked into the container image. Otherwise
you'd build a separate image per environment (dev/staging/prod) and rebuild+redeploy for every
setting change. Principle (12-factor app): **separate config from code** → build the image once,
feed it different config per environment.

## ConfigMap (non-sensitive config)
A named bag of key-value pairs. Pods consume it as **env vars** or as **mounted files**.
The Pod stays generic; the ConfigMap supplies environment-specific values — change config
without rebuilding the image.

```bash
kubectl create configmap app-config \
  --from-literal=APP_COLOR=blue --from-literal=APP_MODE=production
kubectl get configmap app-config -o yaml
```
Injected via `envFrom: [{configMapRef: {name: app-config}}]`. Verified inside the Pod:
`kubectl exec config-demo -- env | grep APP` → `APP_COLOR=blue`, `APP_MODE=production` (values
that don't exist in the base image = injected at runtime).

## Secret (sensitive data) — SAME mechanics, big gotcha
```bash
kubectl create secret generic db-secret --from-literal=DB_PASSWORD='<your-password>'
kubectl get secret db-secret -o yaml     # data: DB_PASSWORD: <base64-blob> (encoded, not encrypted)
kubectl get secret db-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 --decode; echo
# -> prints the plaintext password (no key, no permission — base64 is NOT encryption)
```
Injected via `envFrom: [{secretRef: {name: db-secret}}]` alongside the ConfigMap.

## THE GOTCHA (interview favorite)
**A Secret is only base64-ENCODED by default, NOT encrypted.** Proved it: decoded the password
in one command. What actually makes a Secret safer than a ConfigMap:
1. **RBAC** — restrict *who* can read Secrets (the real day-to-day control). In minikube I'm
   cluster-admin so I see everything; in prod `get secrets` is locked to few identities.
2. **Encryption at rest** — enable `EncryptionConfiguration` so etcd stores them encrypted, not
   plain base64. (App still gets plaintext at runtime; this protects storage, not the value.)
3. **Careful handling** — masked in `describe`/logs; kept in tmpfs (RAM) on nodes; delivered
   only to nodes that need them.
4. **External managers** — Vault, cloud Secret Manager, Sealed/External Secrets for production.

**Golden rule:** NEVER commit a Secret's base64 YAML to git — it's a plaintext password. (That's
why this repo has a ConfigMap manifest but no Secret manifest; the Secret is created imperatively.)

## Commands I used
```bash
kubectl create configmap app-config --from-literal=KEY=VALUE ...
kubectl create secret generic db-secret --from-literal=KEY=VALUE ...
kubectl get configmap|secret <name> -o yaml
kubectl get secret <name> -o jsonpath='{.data.KEY}' | base64 --decode
kubectl delete pod <name> && kubectl apply -f pod.yaml   # can't edit env of a running Pod
kubectl exec <pod> -- env | grep -E 'APP|DB'
```

Manifests: [config-demo-pod.yaml](../manifests/05-config/config-demo-pod.yaml) ·
[app-configmap.yaml](../manifests/05-config/app-configmap.yaml) (Secret intentionally not committed)

## Concept I can explain in an interview
**Q: Is a Kubernetes Secret encrypted?** No — base64-encoded only by default, trivially
decoded. It's made secure by RBAC (who can read it), encryption-at-rest (etcd), careful handling,
and/or an external secret manager — not by the encoding.

**Q: ConfigMap vs Secret?** Same injection mechanics (env vars or files); ConfigMap for
non-sensitive config, Secret for sensitive data (base64, RBAC-restrictable, can be encrypted at
rest, kept in memory on nodes).
