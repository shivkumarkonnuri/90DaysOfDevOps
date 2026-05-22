# Day 80 — Helm Project: Multi-Environment Deployment and CI/CD

## Overview
This document covers the complete Helm journey for the AI-BankApp — creating environment-specific values, adding Helm hooks, packaging the chart, and integrating Helm into the GitOps CI/CD pipeline.

**Repository:** `shivkumarkonnuri/AI-BankApp-DevOps-EKS`
**Chart location:** `helm-chart/bankapp/`

---

## Task 1: Environment-Specific Values Files

### The Core Concept
One chart, three environments. Helm deep-merges values files — `values.yaml` provides defaults, environment-specific files override only what changes. Zero duplication.

```
values.yaml (base defaults)
    +
values-prod.yaml (overrides)
    =
Final merged config (what Helm actually deploys)
```

### Environment Comparison

| Setting | Dev | Staging | Prod | Reasoning |
|---------|-----|---------|------|-----------|
| BankApp replicas | 1 (fixed) | 2-3 (HPA) | 2-4 (HPA) | Dev saves cost, Prod needs HA |
| Image tag | `latest` | `v1.2.0` | `v1.2.0` | Dev = fresh code, Prod = stable pinned |
| MySQL storage | 2Gi | 5Gi | 20Gi | Real data grows in prod |
| MySQL memory | 128Mi | 256Mi | 512Mi | Scale with traffic |
| Ollama memory | 1Gi | 2Gi | 2.5Gi | AI model needs more in prod |
| HPA | ❌ disabled | ✅ enabled | ✅ enabled | Dev doesn't need autoscaling |
| Gateway | ❌ disabled | ❌ disabled | ✅ enabled | Only prod faces real traffic |
| StorageClass | standard | gp3 | gp3 | gp3 = better IOPS for prod |

### values-dev.yaml
```yaml
bankapp:
  replicaCount: 1
  image:
    repository: trainwithshubham/ai-bankapp-eks
    tag: "latest"
    pullPolicy: Always
  resources:
    requests:
      memory: "256Mi"
      cpu: "100m"
    limits:
      memory: "512Mi"
      cpu: "250m"
  autoscaling:
    enabled: false

mysql:
  enabled: true
  resources:
    requests:
      memory: "128Mi"
      cpu: "100m"
    limits:
      memory: "256Mi"
      cpu: "250m"
  persistence:
    size: 2Gi
    storageClass: standard

ollama:
  enabled: true
  model: tinyllama
  resources:
    requests:
      memory: "1Gi"
      cpu: "500m"
    limits:
      memory: "1.5Gi"
      cpu: "1000m"
  persistence:
    size: 5Gi
    storageClass: standard

storageClass:
  create: false
```

### values-staging.yaml
```yaml
bankapp:
  replicaCount: 2
  image:
    repository: trainwithshubham/ai-bankapp-eks
    tag: "v1.2.0"
    pullPolicy: IfNotPresent
  resources:
    requests:
      memory: "256Mi"
      cpu: "250m"
    limits:
      memory: "512Mi"
      cpu: "500m"
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 3
    targetCPUUtilization: 75

mysql:
  enabled: true
  resources:
    requests:
      memory: "256Mi"
      cpu: "250m"
    limits:
      memory: "512Mi"
      cpu: "500m"
  persistence:
    size: 5Gi
    storageClass: gp3

ollama:
  enabled: true
  model: tinyllama
  persistence:
    size: 10Gi
    storageClass: gp3

secrets:
  mysqlRootPassword: StagingPass@456
  mysqlUser: root
  mysqlPassword: StagingPass@456

storageClass:
  create: true
```

### values-prod.yaml
```yaml
bankapp:
  replicaCount: 4
  image:
    repository: trainwithshubham/ai-bankapp-eks
    tag: "v1.2.0"
    pullPolicy: IfNotPresent
  resources:
    requests:
      memory: "256Mi"
      cpu: "250m"
    limits:
      memory: "512Mi"
      cpu: "500m"
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 4
    targetCPUUtilization: 70

mysql:
  enabled: true
  resources:
    requests:
      memory: "512Mi"
      cpu: "500m"
    limits:
      memory: "1Gi"
      cpu: "1000m"
  persistence:
    size: 20Gi
    storageClass: gp3

ollama:
  enabled: true
  model: tinyllama
  resources:
    requests:
      memory: "2Gi"
      cpu: "900m"
    limits:
      memory: "2.5Gi"
      cpu: "1500m"
  persistence:
    size: 10Gi
    storageClass: gp3

secrets:
  mysqlRootPassword: ProdSecure@789
  mysqlUser: root
  mysqlPassword: ProdSecure@789

storageClass:
  create: true

gateway:
  enabled: true
```

### Verification
```bash
# Dev — HPA disabled, replicas hardcoded in Deployment
helm template bankapp-dev . -f values-dev.yaml | grep "replicas"
# Output: replicas: 1

# Prod — HPA enabled, Deployment has NO replicas field (HPA owns it)
helm template bankapp-prod . -f values-prod.yaml | grep -i "replica"
# Output: minReplicas: 2 / maxReplicas: 4 (in HPA resource)
```

**Key insight:** When `autoscaling.enabled: true`, the Deployment template intentionally omits the `replicas` field so HPA has full control. If both Deployment and HPA set replicas, they conflict on every `helm upgrade`.

---

## Task 2: Helm Hooks

### The Problem Hooks Solve
Without hooks, Kubernetes creates ALL resources simultaneously on `helm install`. BankApp starts before MySQL is ready → crashes → restarts repeatedly (CrashLoopBackOff).

```
WITHOUT hooks:                    WITH pre-install hook:
BankApp starts ──┐                [Hook job runs FIRST]
MySQL starts  ───┘                Check MySQL port 3306 ready ✅
BankApp crashes! (MySQL not ready) THEN BankApp Deployment created ✅
```

### Pre-Install Hook: `templates/pre-install-job.yaml`
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "bankapp.fullname" . }}-db-ready
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "bankapp.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": pre-install,pre-upgrade
    "helm.sh/hook-weight": "0"
    "helm.sh/hook-delete-policy": before-hook-creation
spec:
  template:
    spec:
      containers:
        - name: db-check
          image: busybox:1.36
          command:
            - /bin/sh
            - -c
            - |
              echo "Waiting for MySQL to be ready..."
              until nc -z {{ include "bankapp.fullname" . }}-mysql 3306; do
                echo "MySQL not ready, retrying in 3s..."
                sleep 3
              done
              echo "MySQL is ready!"
          resources:
            requests: { memory: "32Mi", cpu: "50m" }
            limits: { memory: "64Mi", cpu: "100m" }
      restartPolicy: Never
  backoffLimit: 10
```

### Hook Annotations Explained

| Annotation | Value | Meaning |
|---|---|---|
| `helm.sh/hook` | `pre-install,pre-upgrade` | Run BEFORE install and BEFORE every upgrade |
| `helm.sh/hook-weight` | `"0"` | Ordering — lower number runs first (useful when chaining multiple hooks) |
| `helm.sh/hook-delete-policy` | `before-hook-creation` | Delete old Job before creating new one (Jobs are immutable in k8s) |

### Why `restartPolicy: Never` on the Job Pod?
- `Never` → on failure, Kubernetes creates a **fresh new pod** for each retry
- Each attempt gets its own pod with clean, separate logs
- `backoffLimit: 10` → after 10 failed attempts, Job is marked Failed → Helm aborts the entire release and rolls back
- This is safer than a half-deployed broken state

### Helm Test: `templates/tests/test-connection.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: {{ include "bankapp.fullname" . }}-test
  namespace: {{ .Release.Namespace }}
  annotations:
    "helm.sh/hook": test
spec:
  containers:
    - name: test
      image: busybox:1.36
      command: ['sh', '-c', 'wget -qO- http://{{ include "bankapp.fullname" . }}-service:8080/actuator/health']
  restartPolicy: Never
```

Run after deployment:
```bash
helm test bankapp-dev -n dev
```

### Hook Lifecycle Summary
```
helm install/upgrade
  │
  ├─► pre-install hook runs (db-check Job)
  │     └─► waits for MySQL port 3306
  │
  ├─► All chart resources created (Deployment, Services, HPA...)
  │
  └─► post-install hook (if defined) — e.g. DB migrations
```

---

## Task 3: Package and Version the Chart

### Packaging
```bash
# Lint — catch template errors before packaging
helm lint bankapp/
# Output: 1 chart(s) linted, 0 chart(s) failed ✅

# Package v0.1.0
helm package bankapp/
# Output: Successfully packaged chart → bankapp-0.1.0.tgz

# Verify contents
tar -tzf bankapp-0.1.0.tgz
```

### Version Bump (added hooks = chart structure changed)
```yaml
# Chart.yaml
version: 0.2.0       # Chart structure version (hooks added)
appVersion: "1.1.0"  # App version deployed
```

```bash
helm package bankapp/
ls -lh bankapp-*.tgz
# bankapp-0.1.0.tgz  (original, no hooks)
# bankapp-0.2.0.tgz  (with pre-install hook + test)
```

### Versioning Concept
```
Chart version  → tracks changes to templates/structure (independent)
appVersion     → tracks the application being deployed

Someone on bankapp-0.1.0 still has a working deployment.
They can choose to upgrade to 0.2.0 (with hooks) when ready.
This is how Bitnami and other chart maintainers manage backwards compatibility.
```

### Install from Package
```bash
helm install my-bankapp bankapp-0.2.0.tgz \
  -f bankapp/values-dev.yaml \
  -n bankapp --create-namespace
```

---

## Task 4: Helm in the AI-BankApp GitOps Pipeline

### Current Pipeline (Raw Manifests)
```
Developer pushes code
       ↓
GitHub Actions builds Docker image
       ↓
Tags with git SHA (e.g. abc1234)
       ↓
sed updates k8s/bankapp-deployment.yml  ← fragile string replacement
       ↓
Commits changed file back to repo
       ↓
ArgoCD detects change → syncs raw YAML to EKS
```

**Problem with `sed`:** Blind string replacement — no YAML structure awareness. Format change = silent breakage.

### With Helm in the Pipeline
```
Developer pushes code
       ↓
GitHub Actions builds Docker image
       ↓
Tags with git SHA (e.g. abc1234)
       ↓
yq updates values-prod.yaml  ← structure-aware, safe
       ↓
Commits changed file back to repo
       ↓
ArgoCD detects change → runs helm upgrade on EKS
```

### CI Step with Helm
```yaml
- name: Update Helm values with new image tag
  run: |
    TAG=${{ steps.tag.outputs.sha_short }}
    yq -i '.bankapp.image.tag = "'$TAG'"' helm-chart/bankapp/values-prod.yaml

- name: Commit updated Helm values
  run: |
    git config user.name "github-actions[bot]"
    git config user.email "github-actions[bot]@users.noreply.github.com"
    git add helm-chart/bankapp/values-prod.yaml
    git diff --staged --quiet || git commit -m "ci: update bankapp image to $TAG [skip ci]"
    git push
```

### ArgoCD Application — Raw Manifests vs Helm

```yaml
# Current: raw manifests
source:
  path: k8s

# With Helm:
source:
  path: helm-chart/bankapp
  helm:
    valueFiles:
      - values-prod.yaml
```

**What ArgoCD does with Helm:**
- Runs `helm template` internally on every sync
- Applies the rendered output to the cluster
- Tracks drift against rendered output
- **Never pushes rendered YAML to Git** — Git stays clean with only source templates

### Why `yq` over `sed`?
```bash
# sed — blind string replacement (fragile)
sed -i 's/tag: .*/tag: "'$TAG'"/' values-prod.yaml

# yq — structure-aware YAML editing (safe)
yq -i '.bankapp.image.tag = "'$TAG'"' values-prod.yaml
# Knows YAML hierarchy, never touches wrong fields ✅
```

---

## Task 5: Production Best Practices

### 1. Always Use `helm upgrade --install`
```bash
helm upgrade --install bankapp bankapp/ \
  -f bankapp/values-prod.yaml \
  --set bankapp.image.tag=$GIT_SHA \
  -n bankapp --create-namespace \
  --wait --timeout 300s \
  --atomic
```

| Flag | Purpose |
|---|---|
| `upgrade --install` | Works whether release exists or not — no more install vs upgrade decision |
| `--set image.tag=$GIT_SHA` | Pins to exact git commit — full traceability from pod back to code |
| `--wait` | Helm waits until ALL pods Running before returning success |
| `--timeout 300s` | Fail fast if pods aren't ready in 5 mins — prevents CI hanging |
| `--atomic` | Auto-rollback to last good version if upgrade fails — most critical for prod |

**Why `--atomic` matters most:**
A failed deploy that stays broken is worse than no deploy. `--atomic` ensures the cluster always stays in a known-good state.

### 2. Use `helm diff` Before Every Upgrade
```bash
helm plugin install https://github.com/databus23/helm-diff
helm diff upgrade bankapp bankapp/ -f bankapp/values-prod.yaml
```
Like `git diff` for your live cluster. Review exactly what changes before committing to an upgrade. Never upgrade production blind.

### 3. Resource Quotas Per Namespace
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: {{ include "bankapp.fullname" . }}-quota
  namespace: {{ .Release.Namespace }}
spec:
  hard:
    requests.cpu: "2"
    requests.memory: 4Gi
    limits.cpu: "4"
    limits.memory: 8Gi
```
A runaway pod in `dev` cannot consume resources meant for `prod`. Namespace quotas act as circuit breakers for compute resources.

### 4. Never Store Real Secrets in values.yaml
```
❌ values.yaml → Git → exposed forever (Git history is permanent)

✅ Option 1: GitHub Actions secrets → --set flag at deploy time
✅ Option 2: AWS Secrets Manager + External Secrets Operator
✅ Option 3: Sealed Secrets (encrypted, safe to commit)
```

```yaml
# GitHub Actions pattern
- name: Deploy
  env:
    DB_PASS: ${{ secrets.MYSQL_ROOT_PASSWORD }}
  run: |
    helm upgrade --install bankapp bankapp/ \
      --set secrets.mysqlRootPassword=$DB_PASS
```

---

## Task 6: Helm vs Raw Manifests vs Kustomize

### Approach Comparison

| | Raw Manifests | Helm | Kustomize |
|---|---|---|---|
| **What it is** | Plain YAML files | Template engine + package manager | Overlay/patch system |
| **Multi-env** | Copy-paste files (error-prone) | One chart + values files | Base + patches per env |
| **Dependencies** | Manual | `helm dependency` built-in | Manual |
| **Versioning** | Git only | Chart versions + Git | Git only |
| **Logic** | None | Full (if/else, loops) | None |
| **Learning curve** | Low | Medium | Low-Medium |
| **Best for** | Simple, single-env | Complex apps, fresh start | Patching existing YAML |

### The Mental Model
```
Raw manifests → Write everything for every environment (duplication)
Kustomize     → Write base once, patch only what differs per env
Helm          → Write a template factory, generate per env from values
```

### When to Choose What

**Choose Raw Manifests when:**
- Simple app, single environment
- No multi-env requirements

**Choose Helm when:**
- Building from scratch with multiple services
- Need dependency management (MySQL chart, cert-manager)
- Complex logic (if/else based on environment)
- Want versioned, distributable chart packages

**Choose Kustomize when:**
- You already have 50 raw YAML files and just need env variations
- You want to patch existing manifests without rewriting them
- No templating logic needed — just field overrides per env

### For AI-BankApp
The Helm chart is the right choice — 3 services (BankApp + MySQL + Ollama), HPA, hooks, environment-specific configs, and built from scratch. Kustomize would have made sense if we were patching the existing `k8s/` directory instead.

---

## 3-Day Helm Journey Summary

| Day | Concept | AI-BankApp Connection |
|-----|---------|----------------------|
| 78 | Helm install, repos, values, upgrade, rollback | Deployed MySQL via Bitnami chart |
| 79 | Custom chart from scratch, Go templates | Converted 12 raw `k8s/` manifests into a Helm chart |
| 80 | Multi-env values, hooks, packaging, CI/CD | Production-ready chart with dev/staging/prod configs |

---

## Key Takeaways

1. **One chart, many environments** — values files handle differences, chart handles structure
2. **HPA + Deployment replicas conflict** — when HPA is enabled, remove `replicas` from Deployment
3. **Hooks = lifecycle control** — pre-install ensures dependencies are ready before app starts
4. **`--atomic` is non-negotiable in CI/CD** — always auto-rollback on failure
5. **Never put secrets in Git** — use pipeline secrets + `--set` flag or External Secrets Operator
6. **ArgoCD renders Helm templates internally** — Git stores source, not rendered YAML
7. **`yq` over `sed`** — structure-aware YAML editing is always safer in CI pipelines

---

*Day 80 of #90DaysOfDevOps | #DevOpsKaJosh | TrainWithShubham*
