# Kubernetes Deployment

Run Wiktapi on Kubernetes. The API runs as a long-lived `Deployment`, while the data pipeline (`download` / `import` / `index`) runs as batch `Job`s sharing the same SQLite database.

## Architecture

| Component     | Kind                     | Image                                           |
| ------------- | ------------------------ | ----------------------------------------------- |
| API server    | `Deployment` + `Service` | `ghcr.io/rizqisugiarto/wiktapi.dev:main`        |
| Data pipeline | `Job` / `CronJob`        | `ghcr.io/rizqisugiarto/wiktapi.dev-worker:main` |

Both images are built and pushed to GHCR automatically by the `.github/workflows/ghcr.yml` workflow on every push to `main`.

The API and the worker each mount the **same `PersistentVolumeClaim`** at different paths:

- API: `/data` (reads `/data/wiktionary.db` via the `DATA_PATH` env var)
- Worker: `/app/packages/api/data` (where the pipeline scripts write `wiktionary.db`)

Because both pods reference one claim, they operate on the same SQLite file.

::: warning SQLite is single-writer
SQLite is unsafe over network/`ReadWriteMany` storage shared by multiple pods. Keep the API at `replicas: 1` and use a `ReadWriteOnce` claim (one pod writes at a time).
:::

## Prerequisites

- A Kubernetes cluster (v1.24+) with a default `StorageClass`
- `kubectl` access
- Push access to GHCR so the cluster can pull the images (or configure a pull secret)

## 1. Shared volume

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wiktapi-data
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 50Gi
```

The English edition is ~2.3 GB compressed; the imported SQLite file grows well beyond that, and each additional edition adds more. Start with `50Gi` and resize as needed.

```bash
kubectl apply -f - <<'EOF'
<paste the PVC above>
EOF
```

## 2. API Deployment and Service

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wiktapi
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wiktapi
  template:
    metadata:
      labels:
        app: wiktapi
    spec:
      containers:
        - name: api
          image: ghcr.io/rizqisugiarto/wiktapi.dev:main
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 3000
          env:
            - name: DATA_PATH
              value: /data/wiktionary.db
          volumeMounts:
            - name: data
              mountPath: /data
          resources:
            requests:
              cpu: 250m
              memory: 512Mi
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: wiktapi-data
---
apiVersion: v1
kind: Service
metadata:
  name: wiktapi
spec:
  selector:
    app: wiktapi
  ports:
    - port: 80
      targetPort: 3000
```

```bash
kubectl apply -f - <<'EOF'
<paste the manifests above>
EOF
```

## 3. Initial data load

The volume starts empty. Run a one-shot `Job` to download, import, and index the first data:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: wiktapi-init
spec:
  backoffLimit: 0
  ttlSecondsAfterFinished: 3600
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: worker
          image: ghcr.io/rizqisugiarto/wiktapi.dev-worker:main
          command: ["sh", "-c"]
          args:
            - pnpm run download -- --editions en && pnpm run import && pnpm run index
          volumeMounts:
            - name: data
              mountPath: /app/packages/api/data
          resources:
            requests:
              cpu: 500m
              memory: 1Gi
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: wiktapi-data
```

```bash
kubectl apply -f - <<'EOF'
<paste the Job above>
EOF

# Wait for it to finish (can take a while — the download alone is ~2.3 GB)
kubectl wait --for=condition=complete job/wiktapi-init --timeout=2h

# Check the result
kubectl logs job/wiktapi-init
```

Add more editions by listing them: `pnpm run download -- --editions en,fr,de`.

## 4. Scheduled refreshes

kaikki.org publishes updated dumps roughly monthly. A `CronJob` re-runs the pipeline on a schedule. Use the staging pattern so the API keeps serving the old database until the swap.

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: wiktapi-refresh
spec:
  schedule: "0 3 1 * *" # 03:00 UTC on the 1st of each month
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 1
  failedJobsHistoryLimit: 2
  jobTemplate:
    spec:
      backoffLimit: 0
      template:
        spec:
          restartPolicy: Never
          containers:
            - name: worker
              image: ghcr.io/rizqisugiarto/wiktapi.dev-worker:main
              command: ["sh", "-c"]
              args:
                - pnpm run download -- --force && pnpm run import:staging && pnpm run index:staging && pnpm run swap
              volumeMounts:
                - name: data
                  mountPath: /app/packages/api/data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: wiktapi-data
```

```bash
kubectl apply -f - <<'EOF'
<paste the CronJob above>
EOF
```

`import:staging` writes to `wiktionary.db.new`, `index:staging` builds its indexes, and `swap` atomically renames the new file over the live one — no downtime during the import.

::: tip Restart the API after a swap
The API opens the SQLite file once at startup and holds the connection, so it keeps reading the old file until restarted. After a refresh finishes, roll the Deployment:

```bash
kubectl rollout restart deployment/wiktapi
```

To automate this, give the worker's `ServiceAccount` permission to restart deployments and append `kubectl rollout restart deployment/wiktapi` to the CronJob command.
:::

## 5. Exposing the API

Expose the `Service` through an `Ingress`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: wiktapi
spec:
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: wiktapi
                port:
                  number: 80
```

## Verifying your instance

```bash
kubectl port-forward svc/wiktapi 3000:80

# List loaded editions
curl http://localhost:3000/v1/editions

# List all word languages with entry counts
curl http://localhost:3000/v1/languages

# Look up a word
curl "http://localhost:3000/v1/en/word/chat?lang=fr"

# Prefix search
curl "http://localhost:3000/v1/en/search?q=cha&lang=fr"
```

## Production tips

- **Pin image digests** instead of floating `:main` tags for reproducible rollouts: `image: ghcr.io/rizqisugiarto/wiktapi.dev@sha256:…`.
- **Keep the API at one replica.** SQLite does not tolerate multiple pods writing concurrently; scale via the `readReplicas` of a replication setup only if you know what you're doing.
- **Size the volume for headroom.** Resize the PVC and grow the backing PV before disk fills up — SQLite doesn't love running out of space mid-import.
