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

apiVersion: batch/v1
kind: Job
metadata:
  name: run-report-chinagtt
spec:
  backoffLimit: 0
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
