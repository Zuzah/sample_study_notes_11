# Local Rancher/K8s setup — Ubuntu WSL on Windows

Replays the same local validation done on a Mac (`docs/k8s_local_validation.md`), adapted
for Ubuntu WSL on a Windows machine. Every fix below is already known — this is the lean
path, not the discovery process.

## Prereqs (WSL-specific, no Mac equivalent needed)

- Docker Desktop for Windows installed, WSL2 backend enabled.
- Docker Desktop → Settings → Resources → **WSL Integration** → toggle on for your Ubuntu
  distro. Without this, `docker`/`kubectl` inside WSL can't reach Docker Desktop's engine.
- This repo checked out on this branch, inside WSL's own filesystem (not `/mnt/c/...` —
  much faster).

## Steps

| # | Step | Command / where | Confirm |
|---|------|------|---------|
| 1 | Enable Kubernetes | Docker Desktop → Settings → Kubernetes → Enable → Apply | `kubectl get nodes` → `Ready`, from your WSL shell |
| 2 | Start local Rancher | `docker run -d --restart=unless-stopped --name rancher-local -p 8080:80 -p 8443:443 --privileged rancher/rancher:latest` | `docker ps` shows it running |
| 3 | Get bootstrap password | `docker logs rancher-local \| grep "Bootstrap Password"` | — |
| 4 | Log in, set real password | Browser → `https://localhost:8443` | **write it down this time** |
| 5 | Import cluster | Rancher UI → Import Existing → copy the `kubectl apply -f https://localhost:8443/v3/import/<token>.yaml` it gives you | — |
| 6 | Apply it — **don't run that command as-is**, the URL fetch fails on the self-signed cert | `curl -sk https://localhost:8443/v3/import/<token>.yaml -o /tmp/import.yaml && kubectl apply -f /tmp/import.yaml` | — |
| 7 | Check the agent | `kubectl get pods -n cattle-system` | If `CrashLoopBackOff` with `localhost:8443/ping not accessible` in the logs → go to step 8. If `Running` 1/1, skip to 9. |
| 8 | Fix known localhost gotcha | Rancher UI → ☰ → Global Settings → `server-url` → Edit → `https://host.docker.internal:8443` → Save, then **repeat step 6** (re-curl same URL, re-apply) | `kubectl get pods -n cattle-system` → `1/1 Running` |
| 9 | Build the image | `docker build -t fenergo-platform:local .` (from repo root) | — |
| 10 | Smoke test | `docker run --rm fenergo-platform:local python -m app.cli --help` | Should fail complaining about 6 missing `FENERGO_*` vars — that failure *is* the pass |
| 11 | Base manifests | `kubectl apply -f k8s/namespace.yaml && kubectl apply -f k8s/pvc.yaml` | — |
| 12 | Real Secret — **strip inline `#` comments from `.env` first**, or a corrupted value (e.g. tenant ID) causes a 401 that looks like a credentials problem | `grep '^FENERGO_' .env \| sed -E 's/[[:space:]]*#.*$//' > /tmp/clean.env && kubectl create secret generic fenergo-secrets -n fenergo-platform --from-env-file=/tmp/clean.env && rm /tmp/clean.env` | (or create via Rancher's Storage → Secrets → Create form instead — sidesteps this bug entirely) |
| 13 | Run it | `kubectl apply -f k8s/job-run-report.yaml` | `kubectl get pods -n fenergo-platform -w` |
| 14 | If it fails immediately with a mount/configmap error | one-time namespace-creation race, not real: `kubectl delete job run-report-chinagtt -n fenergo-platform && kubectl apply -f k8s/job-run-report.yaml` | — |
| 15 | Confirm | `kubectl logs -n fenergo-platform -l job-name=run-report-chinagtt --tail=20` | last line: `TRANSFORMED - execution_id=...` |

## Optional but worth it

Repeat step 13 by pasting `k8s/job-run-report.yaml`'s contents into Rancher's **Import
YAML** UI instead of `kubectl apply` — that's the specific interface devops uses, and
re-proving it there on Windows/WSL costs nothing extra.

## Why this machine matters more than the Mac did

If this WSL environment sits inside the corporate network, step 15's real Fenergo call is
the first chance to get real signal on the two risks the Mac-based validation couldn't
test at all (`docs/roadmap.md` Days 27–29): corporate-proxy reachability to Fenergo, and
whether the bank's Root CA is trusted for SSL verification. Worth explicitly noting if
step 15 behaves any differently here than it did on the Mac — that difference itself would
be a real, useful data point, not noise.
