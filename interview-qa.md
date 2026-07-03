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
helper must share the main app's network/filesystem (sidecar pattern).
