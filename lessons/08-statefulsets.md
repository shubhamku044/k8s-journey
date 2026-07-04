# Lesson 08 — StatefulSets ✅

## Why Deployments aren't enough
A Deployment treats Pods as **interchangeable** — random names, any Pod ≈ any other, replacements
are brand-new with no dedicated storage. Great for stateless apps. But a **database cluster** needs
Pods that are NOT interchangeable: each has a role (primary vs replica), owns its own data, and
must keep a stable identity so peers can find it.

## What a StatefulSet gives each Pod (that a Deployment doesn't)
| Property | Deployment | StatefulSet |
|----------|-----------|-------------|
| Names | random (`web-abc123-xy`) | stable ordinals: `web-0`, `web-1` |
| Storage | shared or none | **own PVC per Pod**, follows it across restarts |
| Network id | one shared Service VIP | stable per-Pod DNS via a **headless Service** |
| Ordering | all at once | ordered create (0→1→2) & delete (2→1→0) |

Two enabling pieces:
- **Headless Service** (`clusterIP: None`) — no load-balancing VIP; gives each Pod its own DNS
  name (`web-sts-0.web-sts...`) so Pods are individually addressable.
- **`volumeClaimTemplates`** — auto-creates one PVC per Pod (`data-web-sts-0`, `data-web-sts-1`);
  `web-sts-0` always reattaches to `data-web-sts-0`.

## What I did (proved stable identity + storage)
```bash
kubectl apply -f manifests/08-statefulset/web-statefulset.yaml
kubectl get pods -w     # ORDERED: web-sts-0 Running before web-sts-1 starts
kubectl get pods        # stable names: web-sts-0, web-sts-1 (not random)
kubectl get pvc         # per-Pod: data-web-sts-0, data-web-sts-1
kubectl exec web-sts-0 -- sh -c 'echo "I am pod 0" > /usr/share/nginx/html/index.html'
kubectl exec web-sts-1 -- sh -c 'echo "I am pod 1" > /usr/share/nginx/html/index.html'
kubectl delete pod web-sts-0
# a NEW web-sts-0 came back with the SAME name...
kubectl exec web-sts-0 -- cat /usr/share/nginx/html/index.html   # -> "I am pod 0" (reattached to its own volume)
```

Manifest: [web-statefulset.yaml](../manifests/08-statefulset/web-statefulset.yaml)

## When to use which (rule of thumb)
- **Stateless & interchangeable** (web servers, APIs) → **Deployment**.
- **Stateful with per-instance identity** (databases, Kafka, Cassandra, Elasticsearch, Zookeeper)
  → **StatefulSet**.

This is the direct answer to the L6 question "how do many pods write correctly?": you don't share
one volume — a StatefulSet gives each stateful Pod its OWN volume + stable identity, and the app
(e.g. the database) handles replication/consistency between them.

## Concept I can explain in an interview
**Q: Deployment vs StatefulSet — when do you need a StatefulSet?**
A: When Pods are NOT interchangeable and need a lasting identity across restarts: a stable name,
stable network DNS (headless Service), and their own persistent volume (volumeClaimTemplates),
plus ordered startup/scaling. Classic use: database and other clustered stateful systems.
