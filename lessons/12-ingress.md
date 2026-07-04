# Lesson 12 (Advanced) — Ingress ✅

## The problem
NodePort gives each Service a random high port (31158, 30904…), one entry point each, no
hostnames, no TLS. Not how production exposes apps. **Ingress** = a single front door on ports
80/443 that routes HTTP by **host and/or path** to Services, and can terminate **TLS**.

## Resource vs Controller (the key split)
- **Ingress resource** = the routing rules in YAML (host/path → Service). Just data; does nothing alone.
- **Ingress controller** = a running reverse-proxy (nginx/Traefik) that *watches* Ingress resources
  and actually routes traffic. **No controller → rules are ignored.**
- Enable in minikube: `minikube addons enable ingress` (nginx controller in `ingress-nginx` ns).

> Analogy: resource = routing instructions on a card; controller = the receptionist who reads it.

## What I did
```bash
minikube addons enable ingress
kubectl get pods -n ingress-nginx        # ingress-nginx-controller must be 1/1 Running
kubectl apply -f manifests/12-ingress/adminer-ingress.yaml   # rule: adminer.local -> adminer:8080
kubectl get ingress -n capstone
kubectl describe ingress adminer -n capstone   # rule resolved to the real Pod IP; controller "Sync"
```
Access on **Mac + Docker driver** (node IP 192.168.49.2 isn't reachable from host — same quirk as
NodePort in L4):
```bash
minikube tunnel                          # separate terminal, needs sudo; binds ingress to 127.0.0.1:80
curl -H "Host: adminer.local" http://127.0.0.1/     # returned Adminer's login HTML ✅
# browser: echo "127.0.0.1 adminer.local" | sudo tee -a /etc/hosts  then open http://adminer.local
```

## Routing many apps through ONE Ingress (the payoff)
- **By host:** `adminer.local` → adminer, `api.local` → api (separate `host` rules).
- **By path:** `myapp.local/adminer` → adminer, `myapp.local/api` → api (multiple `paths`).
- One IP, one port 80, many apps; add TLS certs here for HTTPS.

Manifest: [adminer-ingress.yaml](../manifests/12-ingress/adminer-ingress.yaml)

## Concept I can explain in an interview
**Q: Ingress vs a Service (NodePort/LoadBalancer)?** A Service exposes ONE app (NodePort = random
port; LoadBalancer = one cloud LB per service). An Ingress is a single L7 HTTP entry point that
routes by host/path to MANY Services and can terminate TLS — but it needs an Ingress controller
running to enforce the rules.

**Q: Resource vs controller?** Resource = the routing rules (data); controller = the proxy that
watches resources and does the routing. Both required.
