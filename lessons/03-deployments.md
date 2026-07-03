# Lesson 03 — Deployments & ReplicaSets ✅

## What it is (my words)
A **Deployment** is a controller whose job is: *"keep N copies of this Pod running, always."*
It's the guardian a bare Pod lacked. The hierarchy:

```
Deployment      → what I talk to; manages the desired state & rollouts
  └─ ReplicaSet → enforces "keep N Pods alive" (DESIRED vs CURRENT)
       └─ Pod   → the disposable running unit  (name: web-<rs-hash>-<pod-id>)
```

## The reconciliation loop (self-healing)
The ReplicaSet holds a **desired** count and constantly compares it to the **actual** number of
running Pods. If they differ, it acts to close the gap:
- Delete a Pod → actual drops below desired → it creates **exactly one** replacement (not the
  whole set — only the difference).
- Each replacement is a brand-new Pod with a new random suffix. Pods are disposable — I care
  *how many* run, never *which* specific ones.

Proven live with `kubectl get pods -w`: killing a Pod showed `Terminating` while a new one went
`Pending → ContainerCreating → Running` within ~3s.

## Imperative vs Declarative
- **Imperative** (`kubectl create deployment`, `kubectl scale`): bark commands one at a time.
  Fine for learning; leaves no record of desired state beyond shell history.
- **Declarative** (`kubectl apply -f file.yaml`): write desired state to a file, commit to git,
  apply. The file is the source of truth. Foundation of GitOps.

## `create` vs `apply`
- `kubectl create` errors if the object already exists; not repeatable.
- `kubectl apply` creates if absent, **patches the diff** if present → idempotent, re-runnable.
  Second apply after editing `replicas` printed `deployment.apps/web configured` and only added
  the missing Pods, leaving existing ones untouched.

## Labels & selector (how a Deployment owns its Pods)
A Deployment owns Pods by **label**, not by name (names are random). `spec.selector.matchLabels`
must match `spec.template.metadata.labels`, or K8s rejects the manifest.

## Commands I used
```bash
kubectl create deployment web --image=nginx
kubectl get deployments && kubectl get replicasets && kubectl get pods
kubectl scale deployment web --replicas=3
kubectl delete pod <name>              # RS recreates one replacement
kubectl delete deployment web
kubectl apply -f manifests/03-deployments/web-deployment.yaml   # declarative
```

Manifest: [manifests/03-deployments/web-deployment.yaml](../manifests/03-deployments/web-deployment.yaml)

## Concept I can explain in an interview
**Q: Why does deleting a Deployment-managed Pod bring it back, but a bare Pod stays dead?**
A: The Deployment's ReplicaSet enforces a desired replica count via a reconciliation loop —
actual < desired triggers a replacement. A bare Pod has no controller watching it, so nothing
restores it.

**Q: create vs apply?** create is imperative and fails if the object exists; apply is
declarative, idempotent, and patches the diff — so the YAML file (in git) is the source of truth.
