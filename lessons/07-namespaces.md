# Lesson 07 — Namespaces ✅

## What it is (my words)
**Namespaces are virtual clusters inside a cluster** — logical partitions that keep groups of
resources separate. My stuff lives in `default`; Kubernetes' own components live in `kube-system`.

Built-in namespaces: `default`, `kube-system` (control-plane components), `kube-public`,
`kube-node-lease`.

## Why they matter
- **Isolation** — separate dev / staging / prod, or team-a / team-b, in ONE cluster.
- **No name collisions** — names are unique only *within* a namespace, so `web` can exist in both
  `dev` and `default` as two separate objects. A resource's identity = **namespace + name**.
- **Resource quotas** — cap CPU/RAM per namespace.
- **RBAC** — grant a team access to only their namespace.

## Namespaced vs cluster-scoped
- **Namespaced:** Pods, Deployments, Services, ConfigMaps, Secrets, PVCs.
- **Cluster-scoped (belong to no namespace):** Nodes, PersistentVolumes, StorageClasses, and
  Namespaces themselves.

## What I did
```bash
kubectl get namespaces
kubectl get pods -n kube-system            # the cluster's own components
kubectl create namespace dev
kubectl create deployment nginx-dev --image=nginx -n dev
kubectl get pods -n dev                    # nginx-dev IS here
kubectl get pods                           # in 'default' — nginx-dev is NOT here (isolation proven)
```
The `-n <ns>` flag tells kubectl which namespace to look in; without it, kubectl uses `default`.

## Handy
```bash
kubectl config set-context --current --namespace=dev   # set default ns (skip typing -n)
kubectl config set-context --current --namespace=default
kubectl delete namespace dev                           # CASCADES: deletes everything inside it
```
Cross-namespace DNS: a Service `web` in `dev` is reachable as `web.dev` /
`web.dev.svc.cluster.local` from other namespaces (same namespace = just `web`).

## Concept I can explain in an interview
**Q: What problem do namespaces solve?** Logical isolation/organization of resources within one
cluster (envs, teams), collision-free naming (unique per namespace, not cluster-wide), plus a
scope for quotas and RBAC.

**Q: Can two namespaces have a deployment with the same name?** Yes — names are unique only within
a namespace; identity is namespace+name. **What's NOT namespaced?** Nodes, PersistentVolumes,
StorageClasses, Namespaces.
