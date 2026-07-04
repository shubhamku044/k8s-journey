# kubectl / minikube Cheatsheet

Grows as I learn. Grouped by task.

## Cluster (minikube)
```bash
minikube start --driver=docker --cpus=2 --memory=2048   # start a capped local cluster
minikube status                                         # health
minikube stop                                           # pause (keep it)
minikube start                                          # resume
minikube delete                                         # nuke it
minikube version
```

## Context / config
```bash
kubectl config current-context     # which cluster kubectl talks to
kubectl config view                # full kubeconfig (API server URL, creds)
kubectl cluster-info               # API server endpoints
```

## Nodes
```bash
kubectl get nodes                  # list nodes + STATUS
kubectl get nodes -w               # watch live (Ctrl+C to stop)
```

## Pods
```bash
kubectl run web --image=nginx      # create a Pod
kubectl get pods                   # list
kubectl get pods -o wide           # + IP, NODE
kubectl describe pod <name>        # full detail; read Events: at the bottom
kubectl logs <name>                # container stdout
kubectl delete pod <name>          # delete (a bare Pod is NOT recreated)
```

## Deployments (self-healing + scaling)
```bash
kubectl create deployment web --image=nginx   # Deployment -> ReplicaSet -> Pod(s)
kubectl get deployments                        # desired vs ready
kubectl get replicasets                        # the "keep N alive" counter (DESIRED/CURRENT/READY)
kubectl get pods                               # names: web-<rs-hash>-<pod-id>
kubectl scale deployment web --replicas=3      # change desired count with one command
kubectl delete pod <name>                      # RS recreates ONE replacement to match desired
```
Rule: delete one Pod under a Deployment → exactly one replacement is created to restore the
desired count. Pods are disposable (new random suffix each time); you care how many, not which.

## Services (stable front door + load balancing)
```bash
kubectl expose deployment web --port=80        # create a ClusterIP Service in front of Pods
kubectl get service web                        # shows CLUSTER-IP (stable virtual IP)
kubectl describe service web                   # Selector: app=web; Endpoints = matching Pod IPs
```
- A Service routes to Pods by **label selector** (`app=web`), NOT by name/IP.
- `Endpoints:` auto-updates as Pods come and go → stable access despite ephemeral Pod IPs.
- Also gives a stable **DNS name** (`web` / `web.default.svc.cluster.local`) = service discovery.

```bash
kubectl expose deployment web --type=NodePort --port=80   # external (dev): node-ip:30000-32767
kubectl get service web                                   # PORT(S): 80:30904/TCP
minikube service web                                      # open NodePort service in browser (mac: keep terminal open)
minikube service web --url                                # just print the URL
```
Service types: **ClusterIP** (internal only) → **NodePort** (dev/test external) →
**LoadBalancer** (production external, cloud). Each stacks on the previous.

## ConfigMaps & Secrets (config separate from image)
```bash
kubectl create configmap app-config --from-literal=APP_COLOR=blue --from-literal=APP_MODE=prod
kubectl create secret generic db-secret --from-literal=DB_PASSWORD='<your-password>'
kubectl get configmap app-config -o yaml
kubectl get secret db-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 --decode; echo  # base64 != encryption
kubectl exec <pod> -- env | grep -E 'APP|DB'      # verify injected env vars
```
Inject in a Pod via `envFrom: [{configMapRef: {name: ...}}, {secretRef: {name: ...}}]`.
Secret is base64-only by default → secure it with RBAC + encryption-at-rest + external managers.
NEVER commit a Secret's YAML to git.

## Volumes & persistent storage
```bash
kubectl apply -f pvc.yaml           # PVC = request for storage
kubectl get pvc                     # STATUS: Bound = matched to a PV
kubectl get pv                      # PV auto-provisioned by the StorageClass
# Pod mounts it: volumes[].persistentVolumeClaim.claimName + containers[].volumeMounts[].mountPath
```
- Container FS is ephemeral (dies with Pod); PV lifecycle is independent → data survives.
- **PV** = real storage, **PVC** = a Pod's claim/request, **StorageClass** = provisions PVs dynamically.
- Access modes: RWO (one node RW) · ROX (many RO) · RWX (many RW, needs NFS/CephFS/cloud file).
- Many concurrent writers → use a **database** (owns concurrency), not a shared volume.
  Stateful pods (DB/Kafka/etc) → **StatefulSet**, each pod its own PVC.

## Namespaces (organize & isolate)
```bash
kubectl get namespaces
kubectl get pods -n kube-system                # peek at cluster components
kubectl create namespace dev
kubectl create deployment nginx-dev --image=nginx -n dev
kubectl get pods -n dev                        # -n selects the namespace (default: 'default')
kubectl config set-context --current --namespace=dev   # set default ns, skip -n
kubectl delete namespace dev                   # CASCADES — deletes everything inside
```
- Names unique per-namespace (identity = namespace+name); same name can exist in two namespaces.
- Namespaced: Pods/Deploy/Svc/CM/Secret/PVC. Cluster-scoped: Nodes/PV/StorageClass/Namespaces.
- Cross-ns DNS: `service.namespace.svc.cluster.local` (or short `service.namespace`).

## Ingress (single HTTP front door — advanced)
```bash
minikube addons enable ingress              # install the nginx Ingress CONTROLLER
kubectl get pods -n ingress-nginx           # controller must be 1/1 Running
kubectl apply -f ingress.yaml               # the Ingress RESOURCE (host/path -> Service rules)
kubectl get ingress -n <ns>
minikube tunnel                             # mac/docker: bind ingress to 127.0.0.1:80 (sudo)
curl -H "Host: adminer.local" http://127.0.0.1/   # test a host rule
```
- **Resource** = routing rules (data); **controller** = the proxy that enforces them. Need both.
- Route many apps via one port 80: by host (`a.local`/`b.local`) or by path (`/a`, `/b`); add TLS here.

## StatefulSets (stable identity + per-Pod storage)
```bash
kubectl apply -f web-statefulset.yaml   # headless Service (clusterIP: None) + StatefulSet
kubectl get pods                        # stable ordinals: web-sts-0, web-sts-1 (not random)
kubectl get pvc                         # one PVC per Pod: data-web-sts-0, data-web-sts-1
kubectl delete pod web-sts-0            # returns as SAME name, reattached to its own volume
```
- StatefulSet = stable name + stable DNS (headless Service) + own PVC (volumeClaimTemplates) + ordered ops.
- Use for NON-interchangeable Pods: databases, Kafka, Cassandra, Elasticsearch, Zookeeper.
- Stateless/interchangeable → Deployment; stateful with per-instance identity → StatefulSet.

## Health checks (probes)
```yaml
livenessProbe:  { httpGet: {path: /, port: 80}, initialDelaySeconds: 5, periodSeconds: 5 }
readinessProbe: { exec: {command: ["cat","/tmp/ready"]}, periodSeconds: 3 }
startupProbe:   { httpGet: {path: /, port: 80}, failureThreshold: 30, periodSeconds: 10 }
```
```bash
kubectl get pods                 # READY column reflects readiness; RESTARTS reflects liveness
kubectl get endpoints <svc>      # readiness gates whether a Pod appears here (gets traffic)
kubectl describe pod <pod>       # Events show "Liveness/Readiness probe failed"
```
- **Liveness** fail → kubelet RESTARTS the container.
- **Readiness** fail → Pod removed from Service endpoints (no traffic), NOT restarted.
- **Startup** → protects slow starters; disables liveness/readiness until it passes.
- Methods: httpGet · tcpSocket · exec. Use readiness (not liveness) for external-dependency blips.

## Autoscaling (HPA) + resources
```bash
minikube addons enable metrics-server                        # HPA needs a metrics source
kubectl set resources deploy X --requests=cpu=200m --limits=cpu=500m  # HPA needs requests
kubectl top pods                                             # live CPU/mem usage
kubectl autoscale deploy X --cpu-percent=50 --min=1 --max=5  # create HPA
kubectl get hpa -w                                           # watch TARGETS % and REPLICAS
```
- **request** = reserved/guaranteed CPU/RAM (scheduling + HPA baseline); `200m` = 0.2 core.
- **limit** = max CPU/RAM (CPU throttled, memory OOM-killed). NOT about HTTP traffic.
- HPA scales Pods (horizontal) to keep a metric near target, between min/max; scale-down has a
  stabilization/cooldown window. Needs requests + metrics-server.


## Handy flags
- `-o wide` — more columns
- `-w` — watch/stream changes
- `-o yaml` — full object as YAML
