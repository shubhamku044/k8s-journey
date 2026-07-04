# Lesson 10 — Autoscaling (HPA) ✅

## What it is
**Horizontal Pod Autoscaler** = automatically adds/removes **Pods** to keep a metric near a target,
so you don't scale by hand. *Horizontal* = more Pods (scale out); *vertical* (VPA) = bigger Pods
(more CPU/RAM each). HPA can scale on **CPU** (used here), memory, or custom metrics.

## Prereq: resource requests & limits (about CPU/RAM, NOT traffic)
- **request** = CPU/RAM **reserved & guaranteed** for the container. The scheduler uses it to place
  the Pod; the HPA measures usage as a % of it. (`cpu: 200m` = 0.2 of a core.)
- **limit** = the **maximum** CPU/RAM allowed. CPU over limit → throttled; memory over limit →
  container OOM-killed.
- ⚠️ Word trap: "request" here has nothing to do with HTTP requests/traffic — it's hardware.

**HPA needs two things:** (1) resource **requests** on the Pods, and (2) a **metrics source**
(`minikube addons enable metrics-server`).

## What I did (watched it scale out under load)
```bash
minikube addons enable metrics-server
kubectl create deployment php-apache --image=registry.k8s.io/hpa-example
kubectl set resources deployment php-apache --requests=cpu=200m --limits=cpu=500m
kubectl expose deployment php-apache --port=80
kubectl top pods                                              # confirm metrics work
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=5
kubectl get hpa -w
# load (second terminal):
kubectl run -it --rm load-generator --image=busybox -- /bin/sh -c \
  "while true; do wget -q -O- http://php-apache; done"
```
Observed:
```
cpu: 114%/50%  REPLICAS 1 → 3     (over target → scale out)
cpu: 138%/50%  REPLICAS 3
cpu: 114%/50%  REPLICAS 3 → 5     (capped at max=5)
cpu:   0%/50%  REPLICAS 5         (load stopped → scales back to 1 after cooldown)
```
Scale-down is delayed by a **stabilization window** to avoid flapping.

Manifest (declarative version): [php-apache-hpa.yaml](../manifests/10-hpa/php-apache-hpa.yaml)

## Concept I can explain in an interview
**Q: What does an HPA need to work, and how does it decide?** It needs resource `requests` on the
Pods and a metrics source (metrics-server). It compares observed usage (e.g. CPU) to a target %
of the request and adjusts replica count between min and max to converge on the target.

**Q: request vs limit?** request = guaranteed reserved CPU/RAM (scheduling + HPA baseline);
limit = hard ceiling (CPU throttled, memory OOM-killed). Unrelated to HTTP traffic.
