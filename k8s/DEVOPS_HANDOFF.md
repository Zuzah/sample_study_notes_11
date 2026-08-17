# run-report — Kubernetes deployment steps

Pre-validated end-to-end (real Fenergo call, real Rancher Import YAML) on a local cluster.
Files: `namespace.yaml`, `secret.template.yaml`, `deploy.yaml`. Ignore every other file in
this folder.

## Two rules

1. **Whatever Dockerfile you use, your built image must pass this before Rancher:**
   `docker run --rm <image>:<tag> python -m app.cli --help` → must print a command list.
   This is what actually failed last deployment (`python` on `PATH` resolved to a
   different interpreter than the one with dependencies installed) — this one command
   catches that regardless of how your Dockerfile differs from this repo's.
2. **No `.env` file, no volume/file-mounted secret.** Real env vars only, via the Secret's
   `envFrom` — the app doesn't read config any other way.

## Steps

| # | Action | Confirm |
|---|--------|---------|
| 1 | Build image, run the smoke test above, push to your registry | command list printed, then image present in registry |
| 2 | Edit `deploy.yaml`: replace `<<< REPLACE_WITH_YOUR_REGISTRY_IMAGE:TAG >>>` with your image | — |
| 3 | Edit `secret.template.yaml`: fill in 6 `REPLACE_ME` values, save as `secret.yaml` (do not commit) | — |
| 4 | Rancher → **Import YAML** → paste `namespace.yaml` → Import | namespace `fenergo-platform` exists |
| 5 | Rancher → **Import YAML** → paste `secret.yaml` → Import | Secret `fenergo-secrets` exists in that namespace |
| 6 | Rancher → **Import YAML** → paste `deploy.yaml` → Import | Workloads → Jobs → `run-report-chinagtt` reaches `1/1` |
| 7 | Click the pod → **View Logs** | last line reads `TRANSFORMED - execution_id=...` |

## If step 7 fails

Not a YAML problem — this exact sequence already succeeded. Two known causes, need your
input not a code fix (`docs/roadmap.md` Days 27–29):
- Corporate proxy blocking egress to Fenergo
- Root CA not trusted by the pod

## Deliberately not included

`--no-sftp` (real SFTP not finalized yet), SQLite-on-PVC not Postgres (not yet
provisioned).
