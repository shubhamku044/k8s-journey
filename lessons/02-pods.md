# Lesson 02 — Pods ✅

## What it is (my words)
Kubernetes doesn't run containers directly — it runs **Pods**. A Pod is the **smallest
deployable unit**: a thin wrapper around **one or more containers** that:
- **share one network identity** — the whole Pod gets a single IP; containers inside talk
  over `localhost`,
- **share storage** (volumes), and
- **live and die together** — scheduled onto the same node as one unit.

99% of the time a Pod holds exactly one container. The wrapper exists so K8s has a consistent
unit to schedule, network, and heal.

> **container** = my process · **Pod** = the Kubernetes citizen that wraps it.

## Commands I used
```bash
kubectl run web --image=nginx      # create a Pod running nginx
kubectl get pods                   # STATUS: ContainerCreating -> Running
kubectl get pods -w                # watch it flip to Running live (Ctrl+C to stop)
kubectl get pods -o wide           # shows the Pod IP and the NODE it landed on
kubectl describe pod web           # full detail; read the Events: section bottom-up
kubectl logs web                   # stdout of the container inside the Pod
kubectl delete pod web             # delete it
```

## What surprised me / gotchas
- The `ContainerCreating` delay **was the image pull**. The Events showed
  `Pulling image "nginx"` → `Successfully pulled ... in 11.874s`. The Pod can't run until the
  image (181 MB) is downloaded onto the node. Re-running with the same image is near-instant
  because it's now cached on the node → this is why image size matters in production.
- The Pod got its **own internal IP** (`10.244.0.4`, from the pod network range). It's
  internal-only — I can't hit it from my Mac's browser yet. That's what **Services** are for.
- The scheduler **placed** the Pod on a node itself (`Node: minikube/192.168.49.2`) — I didn't
  choose. Kubernetes also auto-mounted a service-account token so the Pod could talk to the API.

## Reading the Events section (the key debugging skill)
`kubectl describe pod <name>` → scroll to `Events:` → it's the Pod's life story in order:
`Scheduled → Pulling → Pulled → Created → Started`. When a Pod is stuck, this shows exactly
where (e.g. stuck at `Pulling` with `ErrImagePull` = bad image name). ~80% of debugging.

## Concept I can explain in an interview
**Q: Difference between a container and a Pod? Why would a Pod ever hold two containers?**
A: A container is just the running process. A Pod is the smallest thing Kubernetes schedules —
a wrapper around 1+ containers that share a single IP, storage, and lifecycle. A Pod holds
**two** containers only when a tightly-coupled **helper** must share the main app's network/
filesystem — the **sidecar** pattern (log shipper, proxy) or an **init** container.
**Anti-pattern:** do NOT put an app and its database in one Pod — separate *services* each get
their own Pod (own lifecycle, scaling, storage).

**Two levels of "restart" (don't confuse them):**
- A **container** inside a Pod *can* be restarted in place by the kubelet if it crashes
  (governed by `restartPolicy`; shows up in the `RESTARTS` column).
- A **Pod** is *not* restarted — it's **ephemeral** and gets **replaced** (new name, new IP).

## The delete experiment (aha)
`kubectl delete pod web` → `kubectl get pods` = **No resources found**. A **bare Pod is not
recreated** — nothing owns it, so self-healing doesn't apply. Self-healing comes from a
**controller** (Deployment/ReplicaSet), which is Lesson 3.
