# Lesson 01 — First Cluster

## What it is (my words)
`kubectl` is only a **client** — a "phone." It doesn't run anything itself; it makes API calls
to a Kubernetes **cluster** (specifically the cluster's **API server**). With no cluster
configured, `kubectl` has no one to call.

**minikube** spins up a real, single-node Kubernetes cluster locally, inside a Docker
container (no separate VM). It's vendor-neutral upstream Kubernetes — the same K8s that
EKS/GKE/AKS run — so what I learn here transfers to the cloud unchanged.

## Why orchestration exists
Running many containers by hand doesn't scale: who restarts a crashed one at 3am, scales
copies on traffic spikes, reschedules when a machine dies, or rolls out a new version with
no downtime? Kubernetes does all of that. I **declare** the desired state and it relentlessly
makes reality match. That word — **declarative** — is the core of everything.

## Commands I used
```bash
# install minikube without Homebrew (multi-user perms issue — see gotchas)
curl -L https://storage.googleapis.com/minikube/releases/latest/minikube-darwin-arm64 \
  -o ~/.local/bin/minikube && chmod +x ~/.local/bin/minikube

minikube start --driver=docker --cpus=2 --memory=2048
kubectl config current-context     # -> minikube
kubectl get nodes                  # -> minikube   Ready   control-plane
kubectl config view                # shows the API server URL minikube wrote

# lifecycle
minikube status | stop | start | delete
```

## What surprised me / gotchas
- Before any cluster, `kubectl cluster-info` failed with `dial tcp [::1]:8080: connection
  refused` — because kubeconfig had no cluster, so kubectl fell back to a hard-coded default.
- A freshly created node reports `NotReady` for a few seconds — the kubelet and CNI (network
  plugin) are still initializing. It flips to `Ready` on its own. Honest state reporting.

## Concept I can explain in an interview
**Q: With no cluster, why does `kubectl` try `localhost:8080` and fail?**
A: `kubectl` reads `~/.kube/config` to learn the API server's address + credentials. With no
cluster configured, it falls back to a default guess (`localhost:8080`); nothing is listening
there → connection refused. `minikube start` fixes it by (1) creating the cluster **and**
(2) writing the connection details into kubeconfig, so kubectl now dials the real API server.
