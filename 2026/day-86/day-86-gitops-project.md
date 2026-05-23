# Day 86 — GitOps Project: End-to-End CI/CD Pipeline with AI-BankApp

## Overview

This document covers the complete GitOps pipeline for the AI-BankApp — from a developer pushing code to ArgoCD automatically deploying the new version to EKS. Zero manual intervention from `git push` to production.

**Repo:** https://github.com/shivkumarkonnuri/AI-BankApp-DevOps-EKS  
**Branch:** `feat/gitops`  
**DockerHub:** `shivkumarkonnuri/ai-bankapp-eks`

---

## Complete GitOps Pipeline Diagram

```
[Developer writes code]
        |
        | git push (src/, pom.xml, Dockerfile only)
        v
[GitHub Actions triggered]
        |
        |-- Checkout code
        |-- Set up JDK 21 (Maven cache)
        |-- Build: ./mvnw clean package -DskipTests -B
        |-- Test: ./mvnw test -B (continue-on-error: true)
        |-- Set image tag: git rev-parse --short HEAD  → e.g. 4cd8410
        |-- Login to DockerHub
        |-- Build & push Docker image
        |       shivkumarkonnuri/ai-bankapp-eks:latest
        |       shivkumarkonnuri/ai-bankapp-eks:4cd8410
        |-- sed update k8s/bankapp-deployment.yml
        |       image: shivkumarkonnuri/ai-bankapp-eks:4cd8410
        |-- git commit "ci: update bankapp image to 4cd8410 [skip ci]"
        |-- git push (bot commit, does NOT retrigger pipeline)
        v
[ArgoCD detects new commit within ~3 minutes]
        |
        |-- Compares: cluster has old image, Git has 4cd8410
        |-- Sync status: OutOfSync
        |-- Performs rolling update
        |       new pods start with new image
        |       old pods terminate gracefully
        |-- Health checks pass
        v
[New version live on EKS — zero downtime]
```

---

## GitHub Actions Workflow — Step by Step

### Trigger block — loop prevention

```yaml
on:
  push:
    branches: [feat/gitops]
    paths:
      - 'src/**'
      - 'pom.xml'
      - 'Dockerfile'
  workflow_dispatch:
```

The `paths` filter ensures the pipeline only runs when application code changes — not when `k8s/` manifests change. This prevents an infinite loop: the pipeline itself commits a manifest update at the end, which would otherwise retrigger a new pipeline run. Two safety nets work together:

- `paths` filter — GitHub Actions ignores pushes that don't touch app code
- `[skip ci]` in the bot commit message — CI platform halts all workflows regardless of which workflow files exist

### Build and test

```yaml
- name: Build with Maven
  run: ./mvnw clean package -DskipTests -B

- name: Run tests
  run: ./mvnw test -B
  continue-on-error: true
```

Build and test are split deliberately. The package step skips tests for speed; tests run in a separate step with `continue-on-error: true` so a test failure doesn't block the deploy in this learning environment.

### Image tag strategy

```yaml
- name: Set image tag
  id: tag
  run: echo "sha_short=$(git rev-parse --short HEAD)" >> "$GITHUB_OUTPUT"
```

Uses a 7-character Git SHA as the image tag (e.g. `4cd8410`). This gives exact traceability from a running pod back to the exact commit that produced it. Both `:latest` and `:sha` tags are pushed.

### GitOps write-back — the critical step

```yaml
- name: Update Kubernetes deployment manifest
  run: |
    sed -i "s|image: shivkumarkonnuri/ai-bankapp-eks:.*|image: shivkumarkonnuri/ai-bankapp-eks:${{ steps.tag.outputs.sha_short }}|" \
      k8s/bankapp-deployment.yml

- name: Commit updated manifest
  run: |
    git config user.name "github-actions[bot]"
    git config user.email "github-actions[bot]@users.noreply.github.com"
    git add k8s/bankapp-deployment.yml
    git diff --staged --quiet || git commit -m "ci: update bankapp image to ${{ steps.tag.outputs.sha_short }} [skip ci]"
    git push
```

The `sed` command does a regex replace in-place:
- Delimiter is `|` not `/` because the image name contains `/` characters
- Pattern `.*` matches any existing tag regardless of what it was
- Only the exact `image: shivkumarkonnuri/ai-bankapp-eks:` line is touched

The `git diff --staged --quiet ||` guard skips the commit entirely if nothing changed — prevents a failed `git commit` when the image tag is identical to the previous run.

---

## Pipeline Execution — Day 86

**Trigger:** Footer text change in `src/main/resources/templates/fragments/layout.html`  
**Commit:** `4cd8410` — `feat: add Day 86 signature to footer`  
**Pipeline duration:** 2m 21s  
**Bot commit:** `baf3db1` — `ci: update bankapp image to 4cd8410 [skip ci]`

Manifest change after pipeline:
```yaml
# Before
image: shivkumarkonnuri/ai-bankapp-eks:latest

# After (bot commit baf3db1)
image: shivkumarkonnuri/ai-bankapp-eks:4cd8410
```

ArgoCD rolling update — pods observed:
```
bankapp-9b55667b-v4cqr    0/1  Running      ← new pod starting
bankapp-b7f5f768f-8pcmj   1/1  Running      ← old pod serving traffic
bankapp-9b55667b-v4cqr    1/1  Running      ← new pod passes readiness probe
bankapp-b7f5f768f-8pcmj   1/1  Terminating  ← old pod gracefully killed
bankapp-9b55667b-bn7v5    0/1  Init:0/2     ← second new pod starting
bankapp-9b55667b-bn7v5    1/1  Running      ← second new pod healthy
bankapp-b7f5f768f-r4rzl   1/1  Terminating  ← second old pod killed
```

Zero downtime — new pods passed readiness probes before old pods were terminated.

---

## Drift Detection and Self-Healing

ArgoCD is configured with `selfHeal: true` and `automated prune`. All three drift scenarios were tested.

### Scenario 1 — Unauthorized scale-down

```bash
kubectl scale deployment bankapp -n bankapp --replicas=1
```

**Result:** ArgoCD detected and restored replicas to 2 in under 2 minutes. The `argocd app get` run immediately after the scale already showed `Synced` — self-heal completed before the command returned.

### Scenario 2 — Wrong image injected

```bash
kubectl set image deployment/bankapp bankapp=nginx:latest -n bankapp
```

**Result:** ArgoCD reverted the image change so quickly that nginx pods never appeared in `kubectl get pods -w`. Deployment hash stayed `9b55667b` (correct image) throughout. Self-heal was effectively instant.

**What would happen if `selfHeal` was disabled:** ArgoCD would detect `OutOfSync` and alert, but take no action. The cluster would run nginx indefinitely until a human manually synced. The application would be broken with no automated recovery.

### Scenario 3 — Critical service deleted

```bash
kubectl delete service bankapp-service -n bankapp
```

**Result:** Service recreated in 5 seconds with a new ClusterIP (`172.20.224.158`). Application traffic routing restored automatically.

### Summary

| Scenario | Action | Recovery time |
|---|---|---|
| Scale down | `replicas=1` | ~2 minutes |
| Wrong image | `nginx:latest` | Instant (pods never spawned) |
| Deleted service | `delete service` | 5 seconds |

---

## ArgoCD Sync History

```
ID  DATE                           REVISION
0   2026-05-23 12:27:35 +0000 UTC  feat/gitops (9f18632)
1   2026-05-23 12:41:05 +0000 UTC  feat/gitops (9f18632)
2   2026-05-23 12:42:50 +0000 UTC  feat/gitops (6ba9c8a)
3   2026-05-23 15:14:47 +0000 UTC  feat/gitops (7df2b10)
4   2026-05-23 15:31:23 +0000 UTC  feat/gitops (7f4e015)
5   2026-05-23 15:33:24 +0000 UTC  feat/gitops (6ba9c8a)
6   2026-05-23 15:34:16 +0000 UTC  feat/gitops (7f4e015)
7   2026-05-23 16:42:11 +0000 UTC  feat/gitops (b636179)
8   2026-05-23 16:51:28 +0000 UTC  feat/gitops (baf3db1)  ← Day 86 deploy
```

Entry 8 (`baf3db1`) is the Day 86 pipeline — the bot commit containing the `4cd8410` image tag.

---

## Full DevOps Pipeline Map

Every block in the 90-day challenge connects to this pipeline:

```
[Developer writes code]
        |
[Git push to GitHub]          ← Day 22-28: Git & GitHub
        |
[GitHub Actions CI]           ← Day 40-49: GitHub Actions
        |-- Build with Maven
        |-- Run tests
        |-- Build Docker image ← Day 29-37: Docker
        |-- Push to DockerHub
        |-- Update K8s manifest
        |-- Commit back to Git
        |
[ArgoCD detects change]       ← Day 84-86: GitOps
        |
[ArgoCD syncs to EKS]         ← Day 81-83: EKS
        |-- Rolling update
        |-- Health checks pass
        |-- HPA scales as needed ← Day 78-80: Helm (HPA, values)
        |
[Prometheus scrapes metrics]  ← Day 73-77: Observability
        |-- Grafana dashboards
        |-- Alerts if something breaks
        |
[App live with zero downtime]
```

---

## 3-Day ArgoCD Journey

| Day | What was built |
|-----|---------------|
| 84 | ArgoCD setup, first GitOps deploy, self-healing |
| 85 | Sync waves, rollbacks, App of Apps, notifications, RBAC |
| 86 | Full CI/CD pipeline, code-to-production, drift detection, teardown |

---

## Key Takeaways

**GitOps is pull-based, not push-based.** ArgoCD pulls from Git and applies to the cluster — the cluster is never directly targeted by the CI pipeline. This means the pipeline can't accidentally break production even if it has a bug.

**Git is the single source of truth.** Whatever is in Git is what runs. Manual `kubectl` changes are automatically reverted. The cluster state is always auditable — check Git, know exactly what's deployed.

**The `[skip ci]` + `paths` filter combination is essential.** Without both, the manifest update commit would trigger an infinite pipeline loop. `paths` is the primary guard; `[skip ci]` is the backup that protects against any workflow without a `paths` filter.

**SHA tags over `latest`.** Using `git rev-parse --short HEAD` as the image tag creates a direct link from running pod → Docker image → Git commit. `latest` gives you no traceability.

**Self-healing is production-grade reliability.** All three drift scenarios — scaling, image swap, deleted service — were resolved automatically without any human intervention. In production this means incidents that would previously require an on-call page resolve themselves.

---

## Teardown

```bash
# Delete ArgoCD applications (cascade deletes all K8s resources)
argocd app delete bankapp --cascade -y
argocd app delete monitoring --cascade -y
argocd app delete root-app --cascade -y

# Destroy EKS cluster
cd AI-BankApp-DevOps-EKS/terraform
terraform destroy

# Verify in AWS Console
# - EKS: no clusters
# - EC2: no instances, no load balancers
# - VPC: bankapp-eks VPC deleted
```

The `--cascade` flag is critical — without it ArgoCD deletes the Application resource but leaves all Kubernetes resources (and AWS billing) running.
