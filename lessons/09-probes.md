# Lesson 09 — Health Checks (Probes) ✅

## The problem
"Process running" ≠ "app healthy" ≠ "app ready for traffic." An app can be running but hung
(deadlocked) or still warming up (loading, connecting to DB). Kubernetes can't guess — you tell it
how to check, with **probes**.

## The three probes (and what happens on FAILURE — the key part)
| Probe | Checks | On failure |
|-------|--------|-----------|
| **Liveness** | is it alive/healthy? | kubelet **RESTARTS the container** |
| **Readiness** | ready for traffic? | Pod **removed from Service endpoints** (no traffic), **NOT restarted** |
| **Startup** | finished starting? | holds off liveness/readiness until startup passes (protects slow starters) |

Check methods: **httpGet** (2xx/3xx = pass), **tcpSocket** (port open), **exec** (command exit 0).
Key fields: `initialDelaySeconds`, `periodSeconds`, `failureThreshold`.

## Liveness demo (watched a container restart-loop)
```yaml
args: ["/bin/sh","-c","touch /tmp/healthy; sleep 30; rm -f /tmp/healthy; sleep 600"]
livenessProbe:
  exec: { command: ["cat","/tmp/healthy"] }
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3
```
After 30s the file is gone → 3 failures → Events: `Liveness probe failed` + `will be restarted`.
`RESTARTS` climbs 0→1→2… every ~45s. **Liveness fail = restart.**

## Readiness demo (watched traffic gating, NO restart)
nginx Deployment + Service, `readinessProbe: exec cat /tmp/ready` (file missing at start):
- `READY 0/1`, **`RESTARTS 0`**, `kubectl get endpoints` **empty** → Pod running but excluded
  from the Service (no traffic).
- `kubectl exec <pod> -- touch /tmp/ready` → `READY 1/1`, endpoints now lists the Pod IP → traffic
  flows. **Readiness fail = pulled from Service, never restarted.**

## Real-world judgment: DB connection lost → use READINESS, not liveness
The app is still alive; it just can't serve. Readiness pulls it from traffic and it auto-rejoins
when the DB returns. Liveness would RESTART it — but a restart can't fix an external outage, so
you'd get a restart storm. Rule: **liveness = "broken, restart it"; readiness = "fine but can't
serve, hold traffic." Never restart over a problem a restart can't fix.**

Manifests: [liveness-demo.yaml](../manifests/09-probes/liveness-demo.yaml) ·
[readiness-demo.yaml](../manifests/09-probes/readiness-demo.yaml)

## Concept I can explain in an interview
**Q: Liveness vs readiness vs startup?** Liveness fail → restart the container; readiness fail →
remove from Service endpoints (no traffic) without restarting; startup → gives slow-starting apps
time to boot by disabling the other two until it passes. Use readiness (not liveness) for
temporary inability to serve (e.g. dependency down) to avoid pointless restart loops.
