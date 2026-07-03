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
```

## Handy flags
- `-o wide` — more columns
- `-w` — watch/stream changes
- `-o yaml` — full object as YAML
