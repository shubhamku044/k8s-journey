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


## Handy flags
- `-o wide` — more columns
- `-w` — watch/stream changes
- `-o yaml` — full object as YAML
