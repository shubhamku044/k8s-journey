# Lesson 13 (Advanced) — Helm ✅

## The problem
Raw `kubectl apply` doesn't scale across environments: dev/staging/prod need the same app with
different values (replicas, image tag, hostnames), so you'd copy-paste near-identical manifests →
drift. Complex apps = dozens of manifests to hand-write. **Helm = the package manager for
Kubernetes** (like apt/npm): parameterized, reusable manifest bundles installed with one command.

## The four terms
- **Chart** — the package: a folder of *templated* manifests (+ default values).
- **Template** — a manifest with placeholders, e.g. `replicas: {{ .Values.replicaCount }}`.
- **values.yaml** — defaults that fill the placeholders; override per env with `--set` or a values file.
- **Release** — an installed *instance* of a chart, with a **name** and a **revision history**.
  Install one chart many times → many independent releases.

> one chart + different values = every environment.

## Install (no Homebrew, ~/.local/bin, no sudo)
```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 \
  | HELM_INSTALL_DIR=$HOME/.local/bin USE_SUDO=false bash
helm version   # v3.21.2
```

## What I did
```bash
helm create demo-chart                       # scaffold a full working chart (nginx)
helm template web ./demo-chart               # render templates locally (values filled in)
helm install web ./demo-chart -n helm-demo --create-namespace
helm list -n helm-demo                        # release 'web', REVISION 1
helm upgrade web ./demo-chart -n helm-demo --set replicaCount=3   # REVISION 2, 3 pods (no YAML edit)
helm history web -n helm-demo                 # see revisions
helm rollback web 1 -n helm-demo              # one-command undo → creates REVISION 3 (=state of 1)
helm uninstall web -n helm-demo               # removes the whole release at once
```

Chart artifact: [manifests/demo-chart/](../manifests/demo-chart) (Chart.yaml, values.yaml, templates/).

## Why Helm beats raw kubectl for multi-env
- **Templating**: one chart, different `values` per environment — no duplicated manifests.
- **Releases + revisions**: named installs with full history; `helm rollback` undoes a bad deploy
  in one command (kubectl apply can't do this).
- **Reuse**: install community charts (Postgres, Prometheus) without writing dozens of manifests.

## Concept I can explain in an interview
**Q: chart vs values vs release?** Chart = the templated package; values = the parameters that fill
the templates (overridable per env); release = a named installed instance of a chart with a
revision history.
**Q: Helm vs raw kubectl?** Helm adds templating (one chart → many envs), release management, and
`upgrade`/`rollback` with version history — none of which raw `kubectl apply` provides.
