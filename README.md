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
