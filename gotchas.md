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

## 4. `exec` probe fails because the binary isn't in the image
**Symptom:** Pod stuck `0/1`; Events show the exec probe command failing (e.g. `curl: not found`).
**Cause:** `exec` probes run *inside* the container, so the command's binary must exist in the
image. Minimal images (like `adminer`) often lack `curl`.
**Fix:** Use `httpGet` (or `tcpSocket`) — the kubelet runs those itself, no in-container binary needed.

## 5. Over-aggressive liveness probe crash-loops a healthy app
**Symptom:** Pod keeps restarting; Events: `Liveness probe failed: context deadline exceeded
(Client.Timeout exceeded while awaiting headers)`, then `will be restarted`. But the app logs show
it started fine and the port is listening.
**Cause:** httpGet's default `timeoutSeconds` is **1s**. A slow/single-threaded server (adminer's
PHP dev server) can take longer, so the probe times out and liveness kills the healthy Pod.
**Fix:** raise `timeoutSeconds` (e.g. 5) and use a lightweight **`tcpSocket`** liveness probe;
keep the meaningful httpGet check on **readiness**. Pattern: lightweight liveness, meaningful readiness.
