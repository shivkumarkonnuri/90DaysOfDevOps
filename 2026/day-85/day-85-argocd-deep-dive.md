# Day 85 — ArgoCD Deep Dive: Sync Strategies, Rollbacks, and Multi-App Management

## Overview

Today covers production-grade ArgoCD patterns used by teams managing real Kubernetes clusters at scale.

**Mental Model:** ArgoCD is like an **air traffic control system** for your Kubernetes cluster — controlling *when* things sync, *in what order*, *who can do what*, and *how to recover* from problems.

| Concept | Real-World Analogy |
|---|---|
| Sync Strategies | Manual vs autopilot landing |
| Sync Waves | Runway sequencing (taxiway → runway → gate) |
| Rollbacks | Returning to a previous flight plan |
| App of Apps | One control tower managing all flights |
| Projects + RBAC | Different clearance levels for pilots |

---

## Task 1: Sync Strategies — Automated vs Manual

### The Core Difference

```
Automated Sync = ArgoCD acts as soon as Git changes (within 3 minutes)
Manual Sync    = ArgoCD detects drift but WAITS for human approval
```

### Automated Sync

```yaml
syncPolicy:
  automated:
    prune: true      # Delete resources removed from Git
    selfHeal: true   # Revert manual cluster changes
```

- Best for: **dev/staging** — fast feedback loop
- Risk: a bad commit deploys automatically, even at 3am

### Manual Sync

```yaml
syncPolicy: {}  # No automated section
```

- Best for: **production** — human review gate before anything reaches the cluster
- ArgoCD detects drift and shows `OutOfSync` but will NOT apply changes
- A human must explicitly approve via CLI or UI

### Commands Used

```bash
# Switch to manual sync
argocd app set bankapp --sync-policy none

# Verify the change
argocd app get bankapp | grep -i "sync policy"
# Output → Sync Policy: Manual

# See EXACTLY what differs between cluster and Git (like git diff for your cluster)
argocd app diff bankapp

# Dry run — preview what WOULD change without applying anything
argocd app sync bankapp --dry-run

# Actually sync (human-approved)
argocd app sync bankapp

# Switch back to automated
argocd app set bankapp --sync-policy automated --self-heal --auto-prune
```

### What --dry-run Shows

From the actual output during this task:

```
ConfigMap    bankapp    bankapp-config    OutOfSync   ← our change (LEARNING_DAY key)
Gateway      bankapp    bankapp-gateway   OutOfSync   Degraded  ← pre-existing issue
HTTPRoute    bankapp    bankapp-route     OutOfSync   Degraded  ← pre-existing issue
```

The dry-run correctly isolated our intentional change (ConfigMap) from
pre-existing issues (Gateway/HTTPRoute). This is exactly why `--dry-run`
is powerful in production — you see impact before committing.

### Key Insight

> Why use manual sync in production even though it's slower?
> **Accountability + safety gate.** Automated sync means a bad commit deploys
> without intent. Manual sync means nothing reaches production without a
> conscious human decision. The slowdown is the feature.

---

## Task 2: Sync Waves — Ordered Deployment

### The Problem Without Ordering

By default, ArgoCD syncs ALL resources in parallel. This causes race conditions:

```
BankApp pod starts     → tries to connect MySQL → MySQL not ready → CRASH
PVC gets created       → Namespace doesn't exist yet → ERROR
HPA tries to scale     → Deployment not created yet → FAIL
Result: crashloop hell, failed deployments, unpredictable startup
```

### The Solution — Sync Wave Annotations

```yaml
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "0"   # integer, can be negative
```

- **Negative numbers** run first, then zero, then positive
- Resources in the **same wave** sync in parallel
- ArgoCD waits for each wave to be **Healthy** before starting the next
- It's like building a house: foundation → walls → roof (never out of order)

### AI-BankApp Wave Order Implemented

| Wave | Resources | Files | Reason |
|---|---|---|---|
| -2 | Namespace, StorageClass | `namespace.yml`, `pv.yml` | Infrastructure must exist before anything else |
| -1 | PVCs, ConfigMap, Secret | `pvc.yml`, `configmap.yml`, `secrets.yml` | Configuration must be ready before apps start |
| 0 | MySQL, Ollama, Services | `mysql-deployment.yml`, `ollama-deployment.yml`, `service.yml` | Dependencies must be running before the app |
| 1 | BankApp Deployment | `bankapp-deployment.yml` | App starts only after all dependencies are healthy |
| 2 | HPA | `hpa.yml` | Scaling rules applied only after app is running |

### Annotation Example

```yaml
# k8s/bankapp-deployment.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bankapp
  namespace: bankapp
  annotations:
    argocd.argoproj.io/sync-wave: "1"
```

### Verifying All Annotations Are Set

```bash
grep -r "sync-wave" k8s/
# Output:
# k8s/namespace.yml:       argocd.argoproj.io/sync-wave: "-2"
# k8s/pv.yml:              argocd.argoproj.io/sync-wave: "-2"
# k8s/configmap.yml:       argocd.argoproj.io/sync-wave: "-1"
# k8s/secrets.yml:         argocd.argoproj.io/sync-wave: "-1"
# k8s/pvc.yml:             argocd.argoproj.io/sync-wave: "-1"
# k8s/mysql-deployment.yml: argocd.argoproj.io/sync-wave: "0"
# k8s/ollama-deployment.yml: argocd.argoproj.io/sync-wave: "0"
# k8s/service.yml:         argocd.argoproj.io/sync-wave: "0"  (×3 — three services)
# k8s/bankapp-deployment.yml: argocd.argoproj.io/sync-wave: "1"
# k8s/hpa.yml:             argocd.argoproj.io/sync-wave: "2"
```

### Result After Implementing Waves

```
Wave -2 → Namespace + StorageClass     (waited for Healthy ✅)
    ↓
Wave -1 → PVCs + ConfigMap + Secret    (waited for Healthy ✅)
    ↓
Wave  0 → MySQL + Ollama + Services    (waited for Healthy ✅)
    ↓
Wave  1 → BankApp Deployment           (waited for Healthy ✅)
    ↓
Wave  2 → HPA                          (Done ✅)

Zero crashloops. Zero race conditions. Ordered, predictable deployment.
```

---

## Task 3: Rollbacks

### Understanding ArgoCD Revision History

Every sync is recorded as a revision with a unique ID:

```bash
argocd app history bankapp

# Output:
# SOURCE  https://github.com/shivkumarkonnuri/AI-BankApp-DevOps-EKS.git
# ID   DATE                           REVISION
# 0    2026-05-23 12:27:35 +0000 UTC  feat/gitops (9f18632)  ← initial deploy
# 1    2026-05-23 12:41:05 +0000 UTC  feat/gitops (9f18632)  ← re-sync (self-heal test)
# 2    2026-05-23 12:42:50 +0000 UTC  feat/gitops (6ba9c8a)  ← configmap change
# 3    2026-05-23 15:14:47 +0000 UTC  feat/gitops (7df2b10)  ← sync waves added
```

### Important Discovery: Rollback Requires Manual Sync Mode

```bash
argocd app rollback bankapp 2
# Error: rollback cannot be initiated when auto-sync is enabled
```

**Why this error makes sense:**

```
Auto-sync is ON  →  ArgoCD re-syncs to latest Git every 3 minutes
You rollback     →  cluster goes to old commit
Auto-sync fires  →  ArgoCD immediately syncs BACK to latest
Net result       →  your rollback had zero effect
```

ArgoCD prevents this pointless operation. It's intelligent safety behavior. ✅

### Correct Rollback Procedure

```bash
# Step 1: disable auto-sync first
argocd app set bankapp --sync-policy none

# Step 2: rollback to a specific revision ID
argocd app rollback bankapp 2

# Step 3: check status — will show OutOfSync (expected!)
argocd app get bankapp | grep -E "Sync Status|Health Status|Sync Policy"
# Sync Policy:  Manual
# Sync Status:  OutOfSync from feat/gitops (7f4e015)  ← cluster is at old commit
# Health Status: Degraded
```

### ArgoCD Rollback vs Git Revert

| | ArgoCD Rollback | Git Revert |
|---|---|---|
| What changes | Cluster state only | Git repository |
| Git history | Unchanged | New commit created |
| Sync status after | `OutOfSync` | `Synced` |
| Audit trail in Git | ❌ None | ✅ Full trail |
| Permanent fix | ❌ Temporary | ✅ Yes |
| GitOps correct | ❌ | ✅ |
| Use case | Emergency brake | Proper fix |

### The GitOps-Correct Rollback

```bash
# This is the RIGHT way to rollback in GitOps
git revert HEAD
git push origin feat/gitops
# ArgoCD sees new commit → auto-syncs → cluster updated
# Git history shows:  deploy → revert  (full audit trail)
```

### The Golden Rule

> **ArgoCD rollback = emergency brake** (buys you time, cluster goes back).
> **`git revert` = proper fix** (Git stays as the source of truth, audit trail preserved).
> Always follow up an ArgoCD rollback with a `git revert`.

---

## Task 4: App of Apps Pattern

### The Problem It Solves

```
Without App of Apps:                  With App of Apps:
─────────────────────────────         ──────────────────────────────
kubectl apply each app manually       ONE root app manages ALL apps
30 apps = 30 manual deployments       Add a YAML file = new app deployed
No single source of truth             Git IS the complete cluster inventory
Someone adds app outside Git?         Impossible — root-app controls all
New team member needs setup?          git clone + kubectl apply root-app.yaml
```

### Architecture

```
root-app (watches argocd-apps/ directory in Git)
    │
    ├── bankapp.yaml       → ArgoCD creates bankapp Application
    │                         → syncs from github.com/.../k8s
    │
    ├── monitoring.yaml    → ArgoCD creates monitoring Application
    │                         → syncs from prometheus-community Helm chart
    │
    └── envoy-gateway.yaml → ArgoCD creates envoy-gateway Application
                              → syncs from docker.io/envoyproxy Helm chart
```

### Directory Structure Created

```
argocd-apps/
├── root-app.yaml       ← the parent (bootstrap this manually once)
├── bankapp.yaml        ← child: AI-BankApp
├── monitoring.yaml     ← child: Prometheus + Grafana
└── envoy-gateway.yaml  ← child: Envoy Gateway
```

### Root App Definition

```yaml
# argocd-apps/root-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/shivkumarkonnuri/AI-BankApp-DevOps-EKS.git
    targetRevision: feat/gitops
    path: argocd-apps          # watches this directory
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd          # creates Application objects in argocd namespace
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### Bootstrap Command (Only Manual Step Ever)

```bash
# This is the ONLY command you ever run manually
kubectl apply -f argocd-apps/root-app.yaml
# root-app reads the directory → creates all child Applications → each syncs independently
```

### Result After Applying Root App

```bash
argocd app list

# NAME           CLUSTER                          NAMESPACE             STATUS     HEALTH
# root-app       https://kubernetes.default.svc   argocd                Synced     Healthy
# bankapp        https://kubernetes.default.svc   bankapp               Synced     Healthy
# monitoring     https://kubernetes.default.svc   monitoring            Synced     Healthy
# envoy-gateway  https://kubernetes.default.svc   envoy-gateway-system  OutOfSync  Healthy
```

### Adding a New Application to the Cluster

```bash
# Old way: manual kubectl/helm commands (no audit trail)
helm install cert-manager ...
kubectl apply -f ...

# New way with App of Apps:
cat > argocd-apps/cert-manager.yaml << EOF
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cert-manager
  namespace: argocd
  ...
EOF

git add argocd-apps/cert-manager.yaml
git commit -m "feat: add cert-manager"
git push
# root-app detects new file → creates Application → deploys automatically
```

### Key Observation: App of Apps + Self-Healing

The `argocd.argoproj.io/tracking-id` annotation on the bankapp Application:

```json
"argocd.argoproj.io/tracking-id": "root-app:argoproj.io/Application:argocd/bankapp"
```

This proves root-app **owns** the bankapp Application object. If you manually delete
bankapp from ArgoCD, root-app will recreate it within minutes. Git is the only
source of truth — no manual changes survive.

---

## Task 5: Notifications

### Architecture

```
TRIGGER   → WHEN to notify   (on sync fail, on health degrade, on sync succeed)
TEMPLATE  → WHAT to say      (message format with app name, revision, status)
SERVICE   → WHERE to send    (Slack webhook, email, PagerDuty, generic webhook)
```

Today we configured Triggers and Templates. Adding a real delivery destination
(Slack etc.) just requires adding the Service with a webhook URL.

### Notifications Controller

```bash
kubectl get pods -n argocd -l app.kubernetes.io/component=notifications-controller
# NAME                                               READY   STATUS
# argocd-notifications-controller-5dfc5644c9-cgr7b   1/1     Running
```

Included in modern ArgoCD installations — no separate install needed.

### ConfigMap Configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  # TRIGGERS — define when to fire
  trigger.on-sync-succeeded: |
    - when: app.status.operationState.phase in ['Succeeded']
      send: [app-sync-succeeded]
  trigger.on-sync-failed: |
    - when: app.status.operationState.phase in ['Error', 'Failed']
      send: [app-sync-failed]
  trigger.on-health-degraded: |
    - when: app.status.health.status == 'Degraded'
      send: [app-health-degraded]

  # TEMPLATES — define what message to send
  template.app-sync-succeeded: |
    message: "Application {{.app.metadata.name}} sync succeeded. Revision: {{.app.status.sync.revision}}"
  template.app-sync-failed: |
    message: "Application {{.app.metadata.name}} sync FAILED! Check ArgoCD for details."
  template.app-health-degraded: |
    message: "Application {{.app.metadata.name}} health is DEGRADED. Investigate immediately."
```

### Subscribe an Application to Notifications

```bash
kubectl annotate application bankapp -n argocd \
  notifications.argoproj.io/subscribe.on-sync-succeeded.webhook="" \
  notifications.argoproj.io/subscribe.on-sync-failed.webhook="" \
  notifications.argoproj.io/subscribe.on-health-degraded.webhook=""
```

The `""` is the webhook URL — empty for now, replace with real Slack/webhook URL in production.

### Verify Configuration

```bash
# Check triggers are in ConfigMap
kubectl get configmap argocd-notifications-cm -n argocd -o yaml | grep trigger

# Check subscriptions on the Application
kubectl get application bankapp -n argocd \
  -o jsonpath='{.metadata.annotations}' | python3 -m json.tool
```

### Adding Real Slack Integration

```yaml
# Add to argocd-notifications-cm under data:
service.slack: |
  token: $slack-token
  
# Then subscribe with slack service instead of webhook:
# notifications.argoproj.io/subscribe.on-sync-failed.slack: "deployments-channel"
```

---

## Task 6: Projects and RBAC

### Why Projects Exist

```
Without Projects:
→ Team A can accidentally sync Team B's application
→ Any team can deploy to kube-system (dangerous!)
→ No boundary between teams' workloads
→ One bad deploy can cascade across the entire cluster

With Projects:
→ Team A is locked to bankapp + monitoring namespaces only
→ Deploying to kube-system is rejected by ArgoCD
→ Complete isolation — one team's mistakes cannot affect another
→ Junior devs can sync but NOT rollback (RBAC enforcement)
```

### Creating the bankapp-team Project

```bash
argocd proj create bankapp-team \
  --description "AI-BankApp team project" \
  --src "https://github.com/shivkumarkonnuri/AI-BankApp-DevOps-EKS.git" \
  --dest "https://kubernetes.default.svc,bankapp" \
  --dest "https://kubernetes.default.svc,monitoring"
```

### Project Restrictions Enforced

```bash
argocd proj get bankapp-team

# Name:          bankapp-team
# Description:   AI-BankApp team project
# Destinations:  https://kubernetes.default.svc,bankapp       ✅ allowed
#                https://kubernetes.default.svc,monitoring    ✅ allowed
# Repositories:  https://github.com/shivkumarkonnuri/...      ✅ allowed
#
# All other namespaces (kube-system, argocd, etc.) → BLOCKED
# All other repos → BLOCKED
```

### RBAC Policy

```bash
kubectl patch configmap argocd-rbac-cm -n argocd --patch '
data:
  policy.csv: |
    p, role:bankapp-dev, applications, get, bankapp-team/*, allow
    p, role:bankapp-dev, applications, sync, bankapp-team/*, allow
    p, role:bankapp-dev, applications, rollback, bankapp-team/*, deny
    g, bankapp-developers, role:bankapp-dev
  policy.default: role:readonly
'
```

### RBAC Breakdown

| Permission | bankapp-dev role | Reason |
|---|---|---|
| `get` applications | ✅ allowed | Devs need to see app status |
| `sync` applications | ✅ allowed | Devs can trigger deployments |
| `rollback` applications | ❌ denied | Requires senior engineer approval |
| Everything else | ❌ denied (readonly default) | Least privilege |

### Important Lesson: App of Apps Overrides CLI Changes

When you run `argocd app set bankapp --project bankapp-team`, the root-app's
self-healing immediately **reverts it back** to `default` (as defined in `argocd-apps/bankapp.yaml`).

To make project changes permanent, update the YAML in Git:

```yaml
# argocd-apps/bankapp.yaml
spec:
  project: bankapp-team   # ← change here, not via CLI
```

This is GitOps working correctly — Git is the only source of truth.

### How Projects Prevent Cross-Team Accidents

```
Team A (bankapp-team project):        Team B (payments-team project):
─────────────────────────────         ──────────────────────────────
Can see: bankapp, monitoring          Can see: payments, redis
Can deploy to: bankapp namespace      Can deploy to: payments namespace
Cannot see Team B's apps              Cannot see Team A's apps
Cannot deploy to kube-system          Cannot deploy to kube-system

Result: complete blast radius isolation
```

---

## Summary: Production GitOps Patterns

| Pattern | When to Use | Key Command |
|---|---|---|
| Manual sync | Production deployments needing human approval | `argocd app set <app> --sync-policy none` |
| Automated sync | Dev/staging for fast feedback | `argocd app set <app> --sync-policy automated` |
| `--dry-run` | Before any production sync to preview impact | `argocd app sync <app> --dry-run` |
| Sync waves | Any app with dependencies (databases, config, etc.) | `argocd.argoproj.io/sync-wave: "N"` annotation |
| ArgoCD rollback | Emergency — cluster breaks, need instant recovery | `argocd app rollback <app> <revision-id>` |
| Git revert | Permanent rollback with audit trail | `git revert HEAD && git push` |
| App of Apps | Managing multiple applications at scale | `kubectl apply -f root-app.yaml` (once) |
| Notifications | Alerting on deploy events | Triggers + Templates + Service in ConfigMap |
| Projects + RBAC | Multi-team clusters with access isolation | `argocd proj create` + patch `argocd-rbac-cm` |

## Key Takeaways

1. **Sync strategy is a risk management decision** — automated for speed, manual for safety.

2. **Sync waves eliminate race conditions** — always annotate apps with dependencies.

3. **ArgoCD rollback ≠ GitOps rollback** — ArgoCD rollback is an emergency brake;
   `git revert` is the permanent, auditable fix.

4. **App of Apps = Git as cluster inventory** — the only way to manage dozens of
   apps without chaos. One file in Git = one app in the cluster.

5. **Self-healing enforces GitOps** — any manual change to the cluster (or to ArgoCD
   Application objects) gets reverted. Git always wins.

6. **Projects enforce blast radius isolation** — in multi-team environments, Projects
   are not optional. They prevent the "one bad deploy takes everything down" scenario.

> GitOps is not just "deploy from Git" — it is a complete operational framework
> for managing production Kubernetes with auditability, reliability, and safety.
