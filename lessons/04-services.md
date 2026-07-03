# Lesson 04 — Services & Networking ✅

## The problem
Pods have internal IPs (`10.244.x.x`) that are **ephemeral** — a replaced Pod gets a new IP.
With multiple replicas there's no single IP to target, and hardcoding one breaks on the next
restart. I can't build anything on an address that keeps changing.

## What a Service is (my words)
A **Service** is a stable front door for a set of Pods. It gives:
- a **stable virtual IP** + a **stable DNS name** (`web` / `web.default.svc.cluster.local`), and
- **load-balancing** across all healthy Pods it selects.

It finds its Pods by **label selector** (`selector: app=web`), not by name/IP. Its `Endpoints`
list = the current matching Pod IPs, and it auto-updates as Pods churn. Verified: the Service's
Endpoints exactly matched my 4 Pods' IPs.

## Service types (concentric doors — each opens wider to the outside)
| Type | Reachable from | Use |
|------|----------------|-----|
| **ClusterIP** (default) | inside the cluster only | internal service-to-service (app → DB) |
| **NodePort** | outside via `node-ip:30000-32767` | dev / testing external access |
| **LoadBalancer** | outside via a cloud load balancer + external IP | production on a cloud |

They stack: NodePort builds on ClusterIP; LoadBalancer builds on NodePort.

## The request path (end to end, what I built)
```
Browser
  -> 127.0.0.1:50681       (minikube tunnel on the Mac — docker driver quirk)
    -> 192.168.49.2:30904  (NodePort on the minikube node)
      -> Service web:80     (load-balances by label app=web)
        -> one of 4 nginx Pods :80
```

## Commands I used
```bash
kubectl expose deployment web --port=80                 # ClusterIP (internal only)
kubectl get service web
kubectl describe service web                            # Selector + Endpoints (= Pod IPs)
kubectl delete service web
kubectl expose deployment web --type=NodePort --port=80 # opens node port 30000-32767
kubectl get service web                                 # PORT(S): 80:30904/TCP
minikube service web                                    # opens it in the browser (keep terminal open)
kubectl apply -f manifests/04-services/web-service.yaml # the declarative version
```

Manifest: [manifests/04-services/web-service.yaml](../manifests/04-services/web-service.yaml)

## Gotcha / surprise
- ClusterIP is unreachable from my Mac (`curl 10.106.29.6` just hangs) — it only lives inside
  the cluster. NodePort opened an external door (port 30904 on the node).
- On the **docker driver on macOS**, even the node IP `192.168.49.2` isn't directly reachable,
  so `minikube service` opens a **tunnel** (`127.0.0.1:...`) — the terminal must stay open. This
  is a minikube-on-Mac quirk, not how NodePort behaves on a real node.
- I initially had it backwards: LoadBalancer is for **production** external access, NodePort is
  the **dev/testing** external option. ClusterIP is internal-only.

## Concept I can explain in an interview
**Q: ClusterIP vs NodePort vs LoadBalancer?**
A: ClusterIP = internal-only stable IP for service-to-service. NodePort = exposes the Service on
a high port on every node for external (dev/test) access. LoadBalancer = provisions a cloud load
balancer with a real external IP for production. Each type builds on the previous.

**Q: How does a Service keep working when Pods are replaced?**
A: It targets Pods by label selector, not IP. Its Endpoints list auto-updates as Pods with the
matching label come and go, so the stable Service IP/DNS keeps routing to healthy Pods.
