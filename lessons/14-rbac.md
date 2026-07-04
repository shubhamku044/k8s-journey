# Lesson 14 (Advanced) — RBAC ✅

Closes the loop on the Lesson 5 question: "can anyone read our Secret?" → with RBAC, only whoever
you grant.

## The model (deny-by-default)
Kubernetes grants nothing until you say so. Two kinds of objects, a clean what/who split:
- **Role** = a list of permissions (verbs on resources) *within a namespace*. Answers **what**.
  - `ClusterRole` = cluster-wide version (and for cluster-scoped resources like nodes).
- **RoleBinding** = attaches a Role to a **subject**. Answers **who**.
  - `ClusterRoleBinding` = cluster-wide version.
- **Subject** = a User, Group, or **ServiceAccount** (the identity Pods run as).

Rules are **additive**; there are **no deny rules** — absence of a grant = denied. A Role with no
binding grants nothing.

## What I did
```yaml
# ServiceAccount 'viewer' + Role 'pod-reader' (get/list/watch pods) + RoleBinding linking them,
# all in namespace rbac-demo.
```
```bash
kubectl create namespace rbac-demo
kubectl apply -f manifests/14-rbac/rbac-demo.yaml
SA=system:serviceaccount:rbac-demo:viewer
kubectl auth can-i list pods    --as=$SA -n rbac-demo   # yes  (granted)
kubectl auth can-i get secrets  --as=$SA -n rbac-demo   # no   (never granted → L5 answer)
kubectl auth can-i delete pods  --as=$SA -n rbac-demo   # no   (only get/list/watch)
kubectl auth can-i list pods    --as=$SA -n default     # no   (Role is namespace-scoped)
```
`kubectl auth can-i --as=<subject>` tests permissions AS another identity without switching accounts.

Manifest: [rbac-demo.yaml](../manifests/14-rbac/rbac-demo.yaml)

## Concept I can explain in an interview
**Q: How do Role and RoleBinding work together?** The Role defines a set of permissions (verbs on
resources in a namespace); the RoleBinding grants that Role to a subject (User/Group/ServiceAccount).
Both are required — a Role does nothing until bound.

**Q: Namespaced vs cluster-wide?** Role/RoleBinding are namespace-scoped; ClusterRole/
ClusterRoleBinding are cluster-wide (and needed for cluster-scoped resources like nodes).

**Q: How does RBAC secure Secrets?** Kubernetes is deny-by-default; only identities bound to a role
that permits `get`/`list` on `secrets` can read them — so "anyone can read the Secret" becomes
"only these bound identities can." (Combine with encryption-at-rest from L5.)
