# Lesson 02 — Pods 🚧 (in progress)

## What it is (my words)
Kubernetes doesn't run containers directly — it runs **Pods**. A Pod is the **smallest
deployable unit**: a thin wrapper around **one or more containers** that:
- **share one network identity** — the whole Pod gets a single IP; containers inside talk
  over `localhost`,
- **share storage** (volumes), and
- **live and die together** — scheduled onto the same node as one unit.

99% of the time a Pod holds exactly one container. The wrapper exists so K8s has a consistent
unit to schedule, network, and heal. Occasionally a Pod holds a second helper container next
to the main app (the **sidecar** pattern — e.g. a log shipper).

> **container** = my process · **Pod** = the Kubernetes citizen that wraps it.

## Commands I used
```bash
kubectl run web --image=nginx      # create a Pod running nginx
kubectl get pods                   # STATUS: ContainerCreating -> Running
kubectl get pods -o wide           # shows the Pod IP and the NODE it landed on
kubectl describe pod web           # full detail; read the Events: section bottom-up
kubectl logs web                   # stdout of the container inside the Pod
```

## What surprised me / gotchas
- _(fill in after running: why the ContainerCreating delay? what did the Events section show?)_

## Concept I can explain in an interview
**Q: Difference between a container and a Pod? Why would a Pod ever hold two containers?**
A: _(fill in own words — Pod = schedulable unit wrapping 1+ containers sharing net/storage/
lifecycle; two containers when a helper must share the app's network/filesystem, e.g. sidecar)_

## Next
- Delete the Pod and observe whether K8s recreates it (preview of self-healing → Deployments).
