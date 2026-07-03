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
