# sample_study_notes_11
Python in K8 Notes

# Build steps

1. Create the sqlite db:

```touch reporting_platform.db```

2. Build and migrate just Prod

```bash
docker compose build
docker compose run --rm app alembic upgrade head
```

3. Build with using dev postgres

```bash
docker compose -f docker-compose.yml -f docker-compose-dev.yml up -d postgres
docker compose -f docker-compose.yml -f docker-compose-dev.yml run --rm app alembic current
docker compose -f docker-compose.yml -f docker-compose-dev.yml run --rm app alembic upgrade head

```

5. 


# Deploy steps

```bash
1. Confirm the pod is up
   kubectl get pods -n <namespace>

2. Confirm the DB target (SQLite vs Postgres) and whether a persistent volume is mounted
   kubectl exec -n <namespace> <pod> -- env | grep DATABASE_URL
   kubectl describe pod -n <namespace> <pod> | grep -A5 "Mounts:"
   -> If DATABASE_URL is sqlite and there's NO persistent volume mounted, flag this now —
      execution history will be wiped on every pod restart.

3. Confirm ENV and proxy config are set as expected
   kubectl exec -n <namespace> <pod> -- env | grep -E "^ENV=|CORPORATE_HTTP_PROXY|HTTPX_VERIFY_SSL"

4. Confirm secrets are actually present (don't print values, just check they're set)
   kubectl exec -n <namespace> <pod> -- env | grep -E "FENERGO_CLIENT_ID|SFTP_HOST" 
   (should show the var names populated, not blank)

5. Apply DB migrations inside the pod (skip if already current)
   kubectl exec -n <namespace> <pod> -- alembic current
   kubectl exec -n <namespace> <pod> -- alembic upgrade head

6. Run a real, live report end-to-end (ChinaGTTReport is the only report with real SQL)
   kubectl exec -it -n <namespace> <pod> -- python -m app.cli run-report ChinaGTTReport

i.e kubectl exec -it <pod> -- python -m app.cli run-report ChinaGTTReport

7. Read the output of step 6 carefully
   - "SFTP_COMPLETED" -> full success, skip to step 8
   - "FAILED: CERTIFICATE_VERIFY_FAILED..." -> Root CA not in the pod's trust store
   - "FAILED: ...FenergoProxyError/FenergoUnreachableError..." -> corporate proxy not
     reachable/misconfigured from inside the cluster
   - "FAILED: ...SFTP..." -> pod can't reach the real SFTP host, or SFTP_* creds wrong
   - "FAILED: TRANSFORMED" only, no delivery -> delivery step itself is the problem, not
     Fenergo

8. Confirm the execution actually got recorded in the DB
   kubectl exec -n <namespace> <pod> -- sqlite3 reporting_platform.db \
     "SELECT execution_id, status, output_file_path FROM report_execution ORDER BY created_utc DESC LIMIT 1;"
   (swap for psql if DATABASE_URL is Postgres)

9. Confirm the delivered file actually landed on the real SFTP host's landing zone
   (check via whatever access your team lead has to that host directly)

10. Persistence check — restart the pod and confirm the row from step 8 still exists
    (only do this if it's safe to restart in this environment)
    kubectl delete pod -n <namespace> <pod>
    -> wait for the replacement pod to come up, then repeat step 8's query
    -> if the row is GONE, there's no real persistence and this needs a PVC or real Postgres

```

Job manifist file:

1. Get <IMAGE REF>

```
kubectl get pods -n <namespace>
kubectl get pod <pod-name> -n <namespace> -o jsonpath='{.spec.containers[0].image}'

```

IF its a Deployment instead of bare pod, then do this for image ref

```bash
kubectl get deployment <deployment-name> -n <namespace> -o jsonpath='{.spec.template.spec.containers[0].image}'
```

2. Get <YOUR_APP_SECRET> — the Secret/ConfigMap holding env vars

```bash
kubectl describe pod <pod-name> -n <namespace>
```

Look in the "Environment Variables from:" section of the output — it lists the actual `Secret/ConfigMap` name(s) the container pulls `FENERGO_*/SFTP_*/DATABASE_URL/ENV` from. If it's a Deployment, run the same against `kubectl describe deployment <deployment-name> -n <namespace>` instead.

If there's more than one Secret/ConfigMap (common — e.g. one for non-secret config, one for actual secrets), list all of them; I'll add each as its own envFrom: entry in the file.

Add this file

```yml
# One-shot K8s Job to run ChinaGTTReport (the only report with real, executable SQL today).
# Fill in the two <...> placeholders before applying - everything else is ready to go.
#
# backoffLimit: 0 is intentional, not an oversight - do NOT change this. K8s Jobs retry
# failed pods by default (backoffLimit defaults to 6). A retry of this specific command
# would resubmit to Fenergo from scratch even if the original submission already succeeded
# and only a later step (poll/download/transform/deliver) failed - a known, documented risk
# in this app (see docs/architecture.md Sec 4's CAUTION note). Until resumable execution is
# designed, this Job must fail loudly once and stop, not retry on its own.
#
# ttlSecondsAfterFinished: 3600 - auto-deletes this Job (and its pod) 1 hour after it
# finishes, success or failure, so it doesn't need to be deleted by hand before the next
# run. Pull logs (kubectl logs job/run-report-chinagtt) before that hour is up if you need
# them - once it's cleaned up, the logs go with it.

apiVersion: batch/v1
kind: Job
metadata:
  name: run-report-chinagtt
spec:
  backoffLimit: 0
  ttlSecondsAfterFinished: 3600
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: app
          image: <YOUR_IMAGE_REF>          # e.g. your-registry/fenergo-platform:latest - same image the existing (non-working) pod/deployment uses
          command: ["python", "-m", "app.cli", "run-report", "ChinaGTTReport"]
          envFrom:
            - secretRef:
                name: <YOUR_APP_SECRET>     # whatever Secret/ConfigMap the existing pod spec already references for FENERGO_*/SFTP_*/DATABASE_URL/ENV


```

Run it:

```bash
kubectl apply -f job-run-report.yaml -n <namespace>
```

Confirm it:

```bash
kubectl logs -n <namespace> job/run-report-chinagtt

```

IF Fail, delete it

```bash
kubectl delete job run-report-chinagtt -n <namespace>
```

Aug 16, 2026

# SEtup on your Computer

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




# Deploy DB via rancher:

Here's the same process, done entirely through Rancher's UI — no command line. I'm using "Import YAML" for each resource rather than the structured forms, since that's the reliable path (the structured form is what mangled the command earlier with that comma issue).

## Step 1 — Create Postgres (Secret + storage + Deployment + Service, all at once)

1. In Rancher, go to the correct cluster → project/namespace where the app itself is running.

1. Find the Import YAML button (top-right corner, usually a cloud icon with an up-arrow, or a + — present on almost every screen in Rancher's cluster view).
2. Paste the block below into the text box. Before pasting, replace <PICK-A-PASSWORD> with an actual password (same value used twice — it appears in two places).

```yml
apiVersion: v1
kind: Secret
metadata:
  name: postgres-credentials
type: Opaque
stringData:
  POSTGRES_USER: reporting_platform
  POSTGRES_PASSWORD: <PICK-A-PASSWORD>
  POSTGRES_DB: reporting_platform
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes: ["ReadWriteOnce"]
  resources:
    requests: {storage: 1Gi}
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 1
  selector:
    matchLabels: {app: postgres}
  template:
    metadata:
      labels: {app: postgres}
    spec:
      containers:
        - name: postgres
          image: postgres:16-alpine
          envFrom:
            - secretRef: {name: postgres-credentials}
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: data
          persistentVolumeClaim: {claimName: postgres-pvc}
---
apiVersion: v1
kind: Service
metadata:
  name: postgres
spec:
  selector: {app: postgres}
  ports:
    - port: 5432
```

4. Click Import

## Step 2 — Confirm Postgres is running


5. Go to Workloads → Pods (left navigation). Find the pod named something like `postgres-xxxxxxxxxx`
6. Wait until its status shows Running/Active. Don't move on until it does. 

## Step 3 — Point the app at the new Postgres

7. Go to Storage → Secrets (or wherever Secrets are listed in the left nav — same place <YOUR_APP_SECRET> was created earlier).
8. Open the app's existing Secret (the one already referenced by the run-report Job).
9. Find the DATABASE_URL key and edit its value to exactly

```bash
postgresql+psycopg2://reporting_platform:<SAME-PASSWORD-AS-STEP-1>@postgres:5432/reporting_platform
```

10. Click Save

## Step 4 — Create the database tables (migration)

11.  Go back to Import YAML again. Paste this (replace `<YOUR_IMAGE_REF>` and `<YOUR_APP_SECRET>` with the same values already used in `job-run-report.yaml`):

```yml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
spec:
  backoffLimit: 0
  ttlSecondsAfterFinished: 3600
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: app
          image: <YOUR_IMAGE_REF>
          command: ["alembic", "upgrade", "head"]
          envFrom:
            - secretRef:
                name: <YOUR_APP_SECRET>

```

12.  Click Import

## Step 5 — Confirm the migration succeeded

13. Go to Workloads → Pods, find the pod named db-migrate-xxxxxxxxxx.
14. Click the ⋮ (or "View Logs") button next to it → View Logs.
15. Confirm the output shows migration steps running with no errors (no need to memorize exact text — just confirm there's no red/error output at the end).

## Step 6 — Run the actual report against the new database
16. Go to Workloads → Jobs, find run-report-chinagtt.
17. If it still exists from before, click ⋮ → Delete on it first (Job names can't be reused).
18. Go to Import YAML again and re-paste the original job-run-report.yaml content (same as before, image/secret already filled in).
19. Click Import.
20. Once it finishes, click View Logs on its pod the same way as step 14 — confirm it says SFTP_COMPLETED.


## Step 7 — Confirm the data actually landed in Postgres
21. Go to Workloads → Pods, find the postgres-xxxxxxxxxx pod.
22. Click ⋮ → Execute Shell (opens a terminal inside that pod, in-browser).
23. In that terminal, type:
```bash
psql -U reporting_platform -d reporting_platform -c "SELECT * FROM report_execution ORDER BY created_utc DESC LIMIT 1;"
```

24. Confirm a row comes back with status = SFTP_COMPLETED

# Using SQLLite3

## Steps (Rancher UI, same pattern as before)
If you already changed DATABASE_URL to the Postgres connection string earlier, undo it:

Go to Storage → Secrets, open the app's Secret.
Set DATABASE_URL to: sqlite:///./reporting_platform.db
Click Save.
(If you never touched DATABASE_URL at all, skip this — nothing to do.)

Delete the old Jobs if they still exist (names can't be reused):

Workloads → Jobs → find run-report-chinagtt (and db-migrate if you created it) → ⋮ → Delete on each.
Import YAML (top-right Import YAML button) with this — fill in the same <YOUR_IMAGE_REF> / <YOUR_APP_SECRET> values already used before:

2. Use yml

```
apiVersion: batch/v1
kind: Job
metadata:
  name: run-report-chinagtt
spec:
  backoffLimit: 0
  ttlSecondsAfterFinished: 3600
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: app
          image: <YOUR_IMAGE_REF>
          command: ["sh", "-c", "alembic upgrade head && python -m app.cli run-report ChinaGTTReport"]
          envFrom:
            - secretRef:
                name: <YOUR_APP_SECRET>

```

4. Click Import.

5. Check the result: Workloads → Pods, find run-report-chinagtt-xxxxxxxxxx, click ⋮ → View Logs.

You should see the alembic migration output first, then the pipeline's own log lines, ending in SFTP_COMPLETED.

# Another way

1. Build the app with just docker build, no docker-compose-dev (skipped)

```bash
docker build
# or to tagit
docker build -t <your-image-name>:<tag> .
```

2. Push it to your registry

```yml
docker tag <your-image-name>:<tag> <registry>/<your-image-name>:<tag>
docker push <registry>/<your-image-name>:<tag>
```

3. 6 Keys are required for Python app to work:

```bash
FENERGO_BASE_URL, FENERGO_TENANT_ID, FENERGO_CLIENT_ID, FENERGO_CLIENT_SECRET, FENERGO_TOKEN_URL, FENERGO_SCOPE

```

4. Confirm a PVC is available (Rancher: Storage → PersistentVolumeClaims)

5.  Delete any old Job with the same name (Job names aren't reusable)
   
6.  Import the Job — this combines the SQLite migration (alembic) and the report submission (--no-sftp) into one run

```
apiVersion: batch/v1
kind: Job
metadata:
  name: run-report-chinagtt
spec:
  backoffLimit: 0
  ttlSecondsAfterFinished: 3600
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: app
          image: <YOUR_IMAGE_REF>
          command: ["sh", "-c", "alembic upgrade head && python -m app.cli run-report ChinaGTTReport --no-sftp"]
          env:
            - name: DATABASE_URL
              value: "sqlite:////data/reporting_platform.db"
          envFrom:
            - secretRef:
                name: <YOUR_APP_SECRET>
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: <EXISTING_PVC_NAME>

```

7. Check the result
Rancher: Workloads → Pods → run-report-chinagtt-xxxxxxxxxx → ⋮ → View Logs

# Sun, Aug 16

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

## Files to use

deploy.yml
```yml
# PVC + Job — apply AFTER namespace.yaml and secret.yaml (this Job's Secret reference
# must already exist, and both need the namespace to already exist).
#
# SQLite on a PersistentVolumeClaim (not Postgres) so the DB survives across separate Job
# runs; --no-sftp so this run doesn't depend on real SFTP host/credentials being finalized
# yet; alembic migration combined into the same command as run-report since SQLite's file
# lives in the pod's own filesystem, so a separate migration Job would touch an
# unpersisted, different file unless it shared this exact PVC mount.
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fenergo-data
  namespace: fenergo-platform
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: batch/v1
kind: Job
metadata:
  name: run-report-chinagtt
  namespace: fenergo-platform
spec:
  backoffLimit: 0
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: run-report
          # <<< REPLACE WITH YOUR REGISTRY IMAGE >>> — the only line in this file that
          # should need to change.
          image: "<<< REPLACE_WITH_YOUR_REGISTRY_IMAGE:TAG >>>"
          imagePullPolicy: IfNotPresent
          command: ["sh", "-c"]
          args:
            - "alembic upgrade head && python -m app.cli run-report ChinaGTTReport --no-sftp"
          envFrom:
            - secretRef:
                name: fenergo-secrets
          env:
            - name: DATABASE_URL
              value: "sqlite:////data/reporting_platform.db"
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: fenergo-data

```

job-run-report.yml:

```yml
# Minimal POC K8s shape (docs/decisions.md 2026-08-14): SQLite on a PVC-mounted path
# instead of Postgres, --no-sftp instead of real SFTP delivery, alembic migration combined
# into the same Job command as run-report since SQLite's file lives in the pod's own
# filesystem and a separate migration Job would touch a different, unpersisted file unless
# it shared this same PVC mount.
#
# imagePullPolicy: Never — this image only exists in the local Docker Desktop image cache
# (docker build -t fenergo-platform:local .), never pushed to any registry. Devops's real
# version of this file will reference a real registry image and drop this line (or set
# IfNotPresent/Always per their registry policy).
apiVersion: batch/v1
kind: Job
metadata:
  name: run-report-chinagtt
  namespace: fenergo-platform
spec:
  backoffLimit: 0
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: run-report
          image: fenergo-platform:local
          imagePullPolicy: Never
          command: ["sh", "-c"]
          args:
            - "alembic upgrade head && python -m app.cli run-report ChinaGTTReport --no-sftp"
          envFrom:
            - secretRef:
                name: fenergo-secrets
          env:
            - name: DATABASE_URL
              value: "sqlite:////data/reporting_platform.db"
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: fenergo-data

```

k8s/namespace.yaml:

```yml
apiVersion: v1
kind: Namespace
metadata:
  name: fenergo-platform

```

k8s/pvc.yaml:

```yml
# Backs the SQLite file so it survives across separate Job runs — a fresh pod's local
# filesystem is otherwise wiped every run (docs/decisions.md 2026-08-14). PVC-scoped POC
# shape only; Postgres/Cloud SQL remains the confirmed real production target
# (docs/persistence_strategy.md) once the GKE<->Cloud SQL connection mechanism is resolved.
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fenergo-data
  namespace: fenergo-platform
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi

```

secret.template.yaml"

```yml
# TEMPLATE — fill in the 6 REPLACE_ME values with real Fenergo credentials (presumably
# sourced from Vault), save as secret.yaml, and import that file — NOT this one.
# Do not commit the filled-in version to git.
apiVersion: v1
kind: Secret
metadata:
  name: fenergo-secrets
  namespace: fenergo-platform
type: Opaque
stringData:
  FENERGO_BASE_URL: "REPLACE_ME"
  FENERGO_TENANT_ID: "REPLACE_ME"
  FENERGO_CLIENT_ID: "REPLACE_ME"
  FENERGO_CLIENT_SECRET: "REPLACE_ME"
  FENERGO_TOKEN_URL: "REPLACE_ME"
  FENERGO_SCOPE: "REPLACE_ME"

```

secret.yml:

```yml
# PLACEHOLDER VALUES — safe for local Tier-1 mechanics validation only (proves envFrom
# wiring works), never for a real Fenergo call. Devops must replace every value below with
# the real ones (probably sourced from Vault per docs/TDD.md §5) before using this in the
# real cluster. Do not commit real credentials into this file.
apiVersion: v1
kind: Secret
metadata:
  name: fenergo-secrets
  namespace: fenergo-platform
type: Opaque
stringData:
  FENERGO_BASE_URL: "https://placeholder.example.com"
  FENERGO_TENANT_ID: "placeholder-tenant"
  FENERGO_CLIENT_ID: "placeholder-client"
  FENERGO_CLIENT_SECRET: "placeholder-secret"
  FENERGO_TOKEN_URL: "https://placeholder.example.com/token"
  FENERGO_SCOPE: "placeholder-scope"

```

# Tues, Aug 18

Do this for token

```bash
curl -k --location --request POST \
  "https://lb.vault.nonprod.bns:8200/v1/secret/data/bjh9/gpe-gardengate/ist/b8fb/fenergo-secrets" \
  --header "X-Vault-Token: <VAULT_TOKEN>" \
  --header 'Content-Type: application/merge-patch+json' \
  --header "X-Vault-Namespace: root" \
  --data '{
    "data": {
      "FENERGO_BASE_URL": "<real value>",
      "FENERGO_TOKEN_URL": "<real value>",
      "FENERGO_TENANT_ID": "<real value>",
      "FENERGO_CLIENT_ID": "<real value>",
      "FENERGO_CLIENT_SECRET": "<real value>",
      "FENERGO_SCOPE": "<real value>"
    }
  }'

```

See this throw away yaml workflow:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: ist-fenx-inspector
  namespace: fenx-ist
spec:
  backoffLimit: 0
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: inspector
          image: af.cds.bns:5002/fenx/extractutilitypython:0.0.2
          command: ["sh", "-c", "ls -la /data"]
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: ist-fenx

```

message:

```txt
Attached: deploy.yaml. It has envFrom.secretRef.name: fenergo-secrets expecting a plain K8s Secret synced from our Vault path (bjh9/gpe-gardengate/ist/b8fb/fenergo-secrets). You mentioned it only appears in Rancher once "used in the deployment" — does that mean I need pod annotations on this Job (Vault Agent Injector pattern) instead of envFrom, or a separate manifest (e.g. ExternalSecret/VaultStaticSecret) deployed alongside it? Could you point me to fenx-extractutility-python's manifest as the working example to copy?
```

Final Deploy:

```yml
# Test Report call with Python app using real IST environment — namespace (fenx-ist), PVC (ist-fenx), and Secret
# (fenergo-secrets, synced from Vault) all already exist. This file only creates the Job itself

apiVersion: batch/v1
kind: Job
metadata:
  name: run-report-chinagtt
  namespace: fenx-ist
spec:
  backoffLimit: 0
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: run-report
          # TODO: Replace with image tag
          image: "<<< REPLACE_WITH_YOUR_REGISTRY_IMAGE:TAG >>>"
          imagePullPolicy: IfNotPresent
          command: ["sh", "-c"]
          args:
            - "alembic upgrade head && python -m app.cli run-report ChinaGTTReport --no-sftp"
          envFrom:
            # Rancher's Storage -> Secrets in fenx-ist before relying on this.
            - secretRef:
                name: fenergo-secrets
          env:
            # devops's normal CD pipeline sets this via commonConfigs/ist.yaml — this Job
            # bypasses that pipeline (direct Import YAML)
            - name: ENV
              value: "ist"
            - name: DATABASE_URL
              value: "sqlite:////data/reporting_platform.db"
          volumeMounts:
            # /data is this container's own arbitrary mount point choice, not dictated by
            # anything about the PVC itself — fine to change if devops has a convention.
            - name: data
              mountPath: /data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: ist-fenx
```

# Steps

# Fix: stale `reporting_platform.db` breaking `alembic upgrade head`

| # | Action | Command | Confirm |
|---|--------|---------|---------|
| 1 | Delete the stale db file from the repo | `rm reporting_platform.db` | file gone |
| 2 | Make sure `.dockerignore` excludes `*.db` (add if missing) | check for a `*.db` line in `.dockerignore` | line present |
| 3 | Rebuild the image | `docker build -t af.cds.bns:5002/fenx/extractutilitypython:<next-tag> .` | build succeeds |
| 4 | Push it | `docker push af.cds.bns:5002/fenx/extractutilitypython:<next-tag>` | image in registry |
| 5 | Edit `deploy.yaml` | replace `image:` with the new tag | — |
| 6 | Delete the old Job (it already exists from prior attempts) | Rancher → Jobs → delete `run-report-chinagtt` in `fenx-ist` | job removed |
| 7 | Import the updated `deploy.yaml` | Rancher → **Import YAML** | Job reaches `1/1` |
| 8 | Confirm | click pod → **View Logs** | alembic runs clean — no "table already exists" / "file is not a database" |

Note: if you also test locally via `docker-compose.yml` (separate from the K8s path above), after step 1 run `touch reporting_platform.db` before your next `docker compose run` — that file is bind-mounted there, and Docker creates an empty directory instead of a file if it's missing entirely, which breaks SQLite a different way.


diagnostic_k8s_proof.py:
```python
# Manual diagnostic script, not part of the automated test suite (see tests/ for that) -
# Does not attempt SFTP delivery at all - only proves submit/poll/download/transform.
#
# Run as a module so repo-root imports resolve correctly:
#   python -m poc.diagnostic_k8s_proof                        # defaults to ChinaGTTReport
#   python -m poc.diagnostic_k8s_proof --report-name ChinaGTTReport
#
# Exit code 0 on success, 1 on failure - safe to use as a container's main command.

import argparse
import asyncio
import logging
import sys
from datetime import datetime, timezone

from app.core.config import settings
from app.services.download_service import DownloadService
from app.services.fenergo_service import FenergoService
from app.services.polling_service import poll_until_ready
from app.services.report_definition_service import ReportDefinitionService
from app.services.transform.template_reader import TemplateReader
from app.services.transform.transformation_service import TransformationService

logging.basicConfig(
    level=logging.DEBUG,
    format="%(asctime)s [%(levelname)s] %(name)s: %(message)s",
    stream=sys.stdout,
)
log = logging.getLogger("diagnostic_k8s_proof")


def parse_args() -> argparse.Namespace:
    parser = argparse.ArgumentParser(
        description="Database-free proof that submit/poll/download/transform work "
        "against real Fenergo - see this file's header comment for why this exists."
    )
    parser.add_argument(
        "--report-name",
        default="ChinaGTTReport",
        help="Registered report name. Default: ChinaGTTReport (the only report with ",
    )
    return parser.parse_args()


async def run(report_name: str) -> None:
    fenergo_service = FenergoService()
    download_service = DownloadService()
    transformation_service = TransformationService()

    log.info(f"[1/4] Submitting {report_name} to Fenergo...")
    source = ReportDefinitionService.report_source(report_name)
    submit_result = await fenergo_service.submit(
        source=source, description=f"{report_name} k8s diagnostic proof"
    )
    log.info(f"[1/4] Submitted. report_id={submit_result.report_id}")

    log.info("[2/4] Polling until ready...")
    poll_result = await poll_until_ready(fenergo_service, submit_result.report_id)
    if poll_result.status != "Completed":
        raise RuntimeError(
            f"Polling did not complete: status={poll_result.status} "
            f"error={poll_result.error}"
        )
    log.info("[2/4] Report ready, presigned URL received.")

    log.info("[3/4] Downloading...")
    download_result = await download_service.download(
        presigned_url=poll_result.presigned_url,
        destination_filename=f"{submit_result.report_id}.csv",
    )
    log.info(
        f"[3/4] Downloaded {download_result.size_bytes} bytes to "
        f"{download_result.local_path}"
    )

    log.info("[4/4] Transforming...")
    schema = TemplateReader.parse_template(
        ReportDefinitionService.template_file_path(report_name)
    )
    timestamp = datetime.now(timezone.utc).strftime("%Y%m%d_%H%M%S")
    output_path = settings.download_path / f"{report_name}_diagnostic_{timestamp}.csv"
    transformation_service.transform_file(
        input_path=download_result.local_path,
        output_path=output_path,
        schema=schema,
    )
    log.info(f"[4/4] Transformed output written to {output_path}")

    print(f"PROOF SUCCEEDED: report_id={submit_result.report_id} output={output_path}")


def main() -> None:
    args = parse_args()
    try:
        asyncio.run(run(args.report_name))
    except Exception as exc:
        print(f"PROOF FAILED: {exc}")
        sys.exit(1)


if __name__ == "__main__":
    main()

```

entrypoint.sh:

```sh
#!/bin/sh

echo "Running database-free Fenergo proof (submit/poll/download/transform, no SFTP)..."
python -m poc.diagnostic_k8s_proof --report-name ChinaGTTReport

```

# Aug 19

Update the yml

ist.yml: add

```yml
extraEnvs:
  - name: DATABASE_URL
    value: "sqlite:////app/Audit/reporting_platform.db"
```

change command:

```yml
command:
  # ── OPTION A: current placeholder (what's there now) ──────────────────────
  # Does nothing. Only useful if you want to exec into the pod manually to poke around.
  # - /bin/sh
  # - -c
  # - sleep 3600

  # ── OPTION B: prove the APIs work right now, zero database (recommended next step) ──
  # - python
  # - -m
  # - poc.diagnostic_k8s_proof
  # - --report-name
  # - ChinaGTTReport

  # ── OPTION C: the real, full flow — use once the stale-database issue is fixed ──
  # - /bin/sh
  # - -c
  # - "alembic upgrade head && python -m app.cli run-report ChinaGTTReport --no-sftp"


```




