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
kubectl create secret generic db-secret --from-literal=DB_PASSWORD=SuperSecret123
kubectl get configmap app-config -o yaml
kubectl get secret db-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 --decode; echo  # base64 != encryption
kubectl exec <pod> -- env | grep -E 'APP|DB'      # verify injected env vars
```
Inject in a Pod via `envFrom: [{configMapRef: {name: ...}}, {secretRef: {name: ...}}]`.
Secret is base64-only by default → secure it with RBAC + encryption-at-rest + external managers.
NEVER commit a Secret's YAML to git.


## Handy flags
- `-o wide` — more columns
- `-w` — watch/stream changes
- `-o yaml` — full object as YAML
