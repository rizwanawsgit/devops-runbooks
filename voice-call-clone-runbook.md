

## Linguo API — k3s Staging Operations Runbook

Quick reference guide for managing `linguo-api` on the Contabo `k3s` cluster (`185.183.54.55`).

---
## 1. Environment & Cluster Overview

| Component | Value |
| :--- | :--- |
| **VPS Host** | `185.183.54.55` |
| **Operational User** | `deploy` |
| **Namespace** | `linguo-api` |
| **Deployment / Container** | `linguo-api` / `linguo-api` |
| **Service & NodePort** | `linguo-api` (`80 -> 5000`, `NodePort: 30101`) |
| **ConfigMap** | `linguo-api-config` |
| **Secret** | `linguo-api-secret` |
| **CI/CD Trigger** | Manual (`workflow_dispatch` via GitHub Actions) |

---

## 2. SSH & User Switching (`deploy` ↔ `root`)

### Connect to VPS
```bash
ssh deploy@185.183.54.55

```

### Switch from `deploy` to `root`

Bash

```
sudo -i
# OR
sudo su -

```

### Return from `root` back to `deploy`

Bash

```
exit

```

## 3. Daily Status, Pods & Live Logs

### Check Pods, Services, ConfigMaps & Secrets

Bash

```
kubectl get pods,svc,configmap,secret -n linguo-api -o wide

```

### Live Streaming Logs (All Pods)

Bash

```
kubectl logs -f -n linguo-api -l app=linguo-api --tail=50 --max-log-requests=10

```

### Live Logs with Timestamps

Bash

```
kubectl logs -f -n linguo-api -l app=linguo-api --tail=50 --timestamps --max-log-requests=10

```

### Filter Live Logs for Errors / Specific Keywords

Bash

```
kubectl logs -f -n linguo-api -l app=linguo-api --tail=50 --max-log-requests=10 | grep -E "error|warn|REDIS|GEMINI|FIREBASE" --line-buffered

```

### Logs for a Specific Pod

Bash

```
kubectl logs -f -n linguo-api <pod-name> --tail=50

```

## 4. Check Currently Running Images

### Check Image Configured on Live Deployment

Bash

```
kubectl get deployment linguo-api -n linguo-api -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'

```

### Check Exact Image Running in Each Pod

Bash

```
kubectl get pods -n linguo-api -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[0].image}{"\n"}{end}'

```

### Check Rollout History (Previous Revisions)

Bash

```
kubectl rollout history deployment/linguo-api -n linguo-api

```

## 5. Updating `ConfigMap` & `Secret` Safely

> **Important Rule:** Never commit `linguo-api-secret.yaml` or `.env` files to GitHub. Keep secrets local on the VPS or in `.gitignore`.

### Step 1: Edit & Apply `ConfigMap` (Non-Sensitive Variables)

Bash

```
nano linguo-api-configmap.yaml
kubectl apply -f linguo-api-configmap.yaml

```

### Step 2: Edit & Apply `Secret` (API Keys, DB URLs, Firebase JSON)

Bash

```
nano linguo-api-secret.yaml
kubectl apply -f linguo-api-secret.yaml

```

### Step 3: Restart Pods to Load New Values

Bash

```
kubectl rollout restart deployment/linguo-api -n linguo-api
kubectl rollout status deployment/linguo-api -n linguo-api --timeout=180s

```

> **Why this preserves the latest image:** `kubectl rollout restart` restarts the live cluster deployment in-place using whatever image tag the GitHub Actions pipeline last deployed. Avoid running `kubectl apply -f linguo-api-k8s.yaml` unless you specifically modified the Service or Deployment spec and verified its image tag.

## 6. Validate Firebase Service Account JSON (Before Applying Secret)

If updating `FIREBASE_SERVICE_ACCOUNT` inside `linguo-api-secret.yaml`, run this check before `kubectl apply`:

Bash

```
python3 -c '
import json, yaml
with open("linguo-api-secret.yaml") as f:
    doc = yaml.safe_load(f)
svc = json.loads(doc["stringData"]["FIREBASE_SERVICE_ACCOUNT"])
assert "-----BEGIN PRIVATE KEY-----" in svc["private_key"]
assert "-----END PRIVATE KEY-----" in svc["private_key"]
print("Valid JSON & private_key boundaries OK:", svc["client_email"])
'

```

## 7. Quick Troubleshooting & Rollback

### Describe a Failing / Pending Pod

Bash

```
kubectl describe pod <pod-name> -n linguo-api

```

### Check Cluster Events (Sorted by Time)

Bash

```
kubectl get events -n linguo-api --sort-by='.lastTimestamp'

```

### Test Local Service Connectivity on VPS

Bash

```
curl -i [http://127.0.0.1:30101/health](http://127.0.0.1:30101/health)

```

### Rollback to Previous Deployment Revision (If Bad Deploy Happens)

Bash

```
kubectl rollout undo deployment/linguo-api -n linguo-api
kubectl rollout status deployment/linguo-api -n linguo-api --timeout=180s

```
