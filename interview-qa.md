# Interview Q&A

Concepts in my own words. Add one after every lesson.

### Q: What is `kubectl` and what does it talk to?
`kubectl` is the Kubernetes **command-line client**. It doesn't run workloads itself — it
sends requests to a cluster's **API server**, which is the front door to the whole control
plane. It finds the API server (URL + credentials) from `~/.kube/config`.

### Q: No cluster configured — why does `kubectl` fail at `localhost:8080`?
kubeconfig has no cluster, so kubectl falls back to a hard-coded default endpoint. Nothing
listens there → connection refused. `minikube start` creates a cluster **and** writes the
kubeconfig context, so kubectl then reaches the real API server.

### Q: What does "declarative" mean in Kubernetes?
I describe the **desired state** (e.g. "5 replicas of this image running"), and Kubernetes
continuously works to make actual state match — restarting, rescheduling, scaling as needed.
I don't script the steps; I declare the goal.

### Q: Why does a freshly started node report `NotReady`?
The kubelet and the CNI network plugin are still initializing. Until pod networking is up, the
node honestly reports it can't run workloads yet. It becomes `Ready` on its own.

### Q: What is a Pod, and how is it different from a container?
A Pod is the smallest unit Kubernetes schedules — a wrapper around one or more containers that
share a network identity (one IP), storage, and lifecycle. A container is just the running
process; the Pod is the K8s object that wraps it. A Pod holds multiple containers only when a
helper must share the main app's network/filesystem (sidecar pattern). An app and its database
do NOT belong in one Pod — separate services each get their own Pod.

### Q: Can you "restart" a Pod?
Two levels, don't confuse them: a **container** inside a Pod can be restarted in place by the
kubelet if it crashes (per `restartPolicy`; visible in the `RESTARTS` column). A **Pod** itself
is not restarted — it's ephemeral and gets **replaced** by a new one (new name, new IP).

### Q: If you delete a Pod, does Kubernetes bring it back?
Only if a controller owns it. A **bare Pod** (created via `kubectl run`) is not recreated —
nothing is watching it. A Pod managed by a **Deployment/ReplicaSet** is recreated automatically,
because the controller enforces the desired replica count. That's where self-healing comes from.

### Q: How does a Deployment self-heal? (the reconciliation loop)
A Deployment creates a ReplicaSet, which holds a **desired** replica count. Its controller
continuously compares desired vs **actual** running Pods. If a Pod dies, actual drops below
desired, so the ReplicaSet creates a replacement to close the gap. It only creates the
*difference* — delete one Pod, exactly one is recreated, not the whole set.

### Q: Deployment vs ReplicaSet vs Pod?
Deployment (what I talk to; manages rollouts) → ReplicaSet (enforces "keep N Pods alive") →
Pod (the disposable running unit, named `web-<rs-hash>-<pod-id>`). Each new Pod gets a fresh
random ID — you care about *how many* are running, never *which* specific Pods.

### Q: Why is a Service necessary — what breaks if you use a Pod's IP directly?
Pod IPs are ephemeral: a Pod that dies is replaced by a new one with a **new IP**, so any
hardcoded Pod IP breaks. And with multiple replicas there's no single IP to target. A Service
provides one **stable virtual IP + DNS name** and **load-balances** across all Pods matching its
**label selector**. Its Endpoints list auto-updates as Pods churn, so clients never notice.

### Q: ClusterIP vs NodePort vs LoadBalancer?
- **ClusterIP** (default): internal-only stable IP; service-to-service (app → DB).
- **NodePort**: exposes the Service on a high port (30000–32767) on every node → external
  dev/testing access.
- **LoadBalancer**: provisions a cloud load balancer with a real external IP → production.
Each builds on the previous (LoadBalancer → NodePort → ClusterIP).

### Q: Why separate config from the container image?
So one image runs everywhere (dev/staging/prod) with environment-specific config supplied at
runtime, and config changes don't require rebuilding/redeploying the image (12-factor app).
ConfigMaps hold non-sensitive config; Secrets hold sensitive data. Both inject as env vars or
mounted files.

### Q: Is a Kubernetes Secret encrypted?
No — by default it's only **base64-encoded**, which is trivially reversible (`base64 --decode`,
no key needed). What secures it: **RBAC** (restrict who can read Secrets), **encryption at rest**
(EncryptionConfiguration encrypts etcd), careful handling (masked in describe/logs, stored in
tmpfs on nodes), and/or an **external secret manager** (Vault, cloud KMS, Sealed Secrets). Never
commit a Secret's base64 YAML to git — it's effectively plaintext.

### Q: PV vs PVC vs StorageClass? Why does PVC data survive a Pod restart?
PV = the actual storage in the cluster; PVC = a Pod's request/claim for storage; StorageClass =
dynamically provisions a PV to satisfy a PVC. The container filesystem is ephemeral (dies with
the Pod), but a PV's lifecycle is independent, so a new Pod mounting the same PVC sees the same
data.

### Q: Can many Pods write to one volume? How do writes stay correct?
By default no — RWO restricts read-write to one node. Multi-node writes need RWX (NFS/CephFS/cloud
file), and even then a filesystem gives no transactions/isolation, so concurrent writers corrupt
data. Real systems keep app Pods **stateless** and route writes to **one database** that owns
concurrency (transactions, locks, isolation). Genuinely stateful workloads use a **StatefulSet**
where each Pod gets its own private PVC.
