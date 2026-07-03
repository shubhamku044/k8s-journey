# Gotchas & Fixes

Real problems I hit and how I solved them. (Interview treasure — these show I can debug.)

## 1. `brew install` fails: `/opt/homebrew` not writable
**Symptom:** `Error: The following directories are not writable by your user: /opt/homebrew ...`
**Cause:** Shared Mac with two users. Homebrew is owned by the *other* (office) user, so my
secondary account can't write `/opt/homebrew`.
**Trap:** Homebrew suggests `sudo chown -R <me> /opt/homebrew` — that would **steal ownership
from the office account and break Homebrew there**. Don't do it.
**Fix:** install the tool as a raw per-user binary into `~/.local/bin` (no sudo, no Homebrew),
and add `~/.local/bin` to PATH in `~/.zshrc`. Touches nothing the other user owns.

## 2. `kubectl` → `dial tcp [::1]:8080: connection refused`
**Cause:** No cluster configured in kubeconfig, so kubectl fell back to a default guess.
**Fix:** `minikube start` — creates the cluster and writes the connection context into
`~/.kube/config`.

## 3. Node shows `NotReady` right after `minikube start`
**Cause:** Checked it seconds after creation — kubelet + CNI still initializing.
**Fix:** Not a bug. Wait a few seconds; `kubectl get nodes` flips to `Ready`.
Watch live with `kubectl get nodes -w`.
