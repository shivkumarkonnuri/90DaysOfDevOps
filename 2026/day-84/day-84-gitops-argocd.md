# Day 84 — GitOps with ArgoCD on EKS

## Overview

On Day 84, the AI BankApp was deployed and managed using **GitOps principles** with **ArgoCD** running on an Amazon EKS cluster. The goal was to replace manual `kubectl apply` deployments with a fully automated, Git-driven deployment pipeline where the cluster state always reflects what is committed in Git.

---

## What is GitOps?

GitOps is a deployment methodology where **Git is the single source of truth** for the desired state of your infrastructure and applications.

| Principle | Description |
|-----------|-------------|
| **Declarative** | Desired state is described in YAML manifests, not scripts |
| **Versioned & Immutable** | Every change is a Git commit — full history and rollback via `git revert` |
| **Pulled Automatically** | ArgoCD pulls from Git; CI pipelines never touch the cluster directly |
| **Continuously Reconciled** | ArgoCD constantly checks: "Does the cluster match Git?" and corrects drift |

### GitOps vs Traditional CI/CD

In traditional CI/CD, pipelines have cluster credentials and push changes via `kubectl apply`. This means no audit trail beyond logs, and manual changes on the cluster go undetected.

In GitOps, **only ArgoCD has cluster credentials**. Developers push to Git. ArgoCD does the rest. The Git commit history is the audit trail.

---

## Environment

- **Cluster:** Amazon EKS (us-west-2)
- **ArgoCD Version:** v3.4.2
- **Branch:** `feat/gitops`
- **Manifest Path:** `k8s/`
- **App Namespace:** `bankapp`

---

## Step 1 — Verify ArgoCD Installation

All 7 ArgoCD pods confirmed running in the `argocd` namespace:

```bash
kubectl get pods -n argocd
```

```
NAME                                                READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                     1/1     Running   0          120m
argocd-applicationset-controller-7bc66679cf-z78l6   1/1     Running   0          120m
argocd-dex-server-8b9d65cd7-6wlnw                   1/1     Running   0          120m
argocd-notifications-controller-5dfc5644c9-cgr7b    1/1     Running   0          120m
argocd-redis-7f54f76887-qdvwd                       1/1     Running   0          120m
argocd-repo-server-797f75479b-6b8kc                 1/1     Running   0          120m
argocd-server-687cbdfc4c-qpgjf                      1/1     Running   0          120m
```

ArgoCD was exposed via an **AWS LoadBalancer**:

```
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP                                    PORT(S)
argocd-server   LoadBalancer   172.20.205.52   aa344b914144f402b8477aa39cf7be77-...elb.amazonaws.com   80,443
```

### ArgoCD CLI Installation

```bash
curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x argocd
sudo mv argocd /usr/local/bin/
argocd version --client
# argocd: v3.4.2
```

### CLI Login

```bash
argocd login <ARGOCD_URL> --username admin --password <PASSWORD> --insecure
# 'admin:login' logged in successfully
```

---

## Step 2 — ArgoCD Application Manifest

The `application.yml` tells ArgoCD what to watch and where to deploy.

```yaml
# argocd/application.yml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: bankapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/shivkumarkonnuri/AI-BankApp-DevOps-EKS.git
    targetRevision: feat/gitops
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: bankapp
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
```

### Field Reference

| Field | Purpose |
|-------|---------|
| `repoURL` | GitHub repo ArgoCD watches for changes |
| `targetRevision` | Branch to track (`feat/gitops`) |
| `path: k8s` | Only watch the `k8s/` directory |
| `destination.server` | Deploy to the same cluster ArgoCD runs in |
| `destination.namespace` | Deploy into the `bankapp` namespace |
| `automated` | Sync automatically on every Git change |
| `prune: true` | Delete cluster resources removed from Git |
| `selfHeal: true` | Revert manual cluster changes back to Git state |
| `CreateNamespace=true` | Auto-create `bankapp` namespace if missing |
| `ServerSideApply=true` | Use server-side apply to avoid field conflicts |

### Apply the Manifest

```bash
kubectl apply -f argocd/application.yml
# application.argoproj.io/bankapp created
```

This is the **last `kubectl apply` ever needed** for the BankApp. All future deployments go through Git.

---

## Step 3 — Application Deployment

After applying the manifest, ArgoCD synced the `k8s/` directory and brought up all pods:

```bash
kubectl get pods -n bankapp
```

```
NAME                        READY   STATUS    RESTARTS   AGE
bankapp-6d69cdf947-2vxhb    1/1     Running   0          12m
bankapp-6d69cdf947-ldjbb    1/1     Running   0          12m
bankapp-6d69cdf947-pplt2    1/1     Running   0          12m
bankapp-6d69cdf947-tjsm8    1/1     Running   0          12m
cm-acme-http-solver-jjhdg   1/1     Running   0          12m
mysql-778d8d585d-kh46b      1/1     Running   0          12m
ollama-cbd479bf4-k7j79      1/1     Running   0          12m
```

All 7 pods healthy — 4× bankapp replicas, MySQL, Ollama, and the cert-manager ACME solver for TLS.

### Sync Status

```bash
argocd app get bankapp --show-operation
```

```
Sync Status:   Synced to feat/gitops (9f18632)
Health Status: Degraded   # gateway only — TLS cert pending
Phase:         Succeeded
Duration:      2s
```

> **Note:** The `bankapp-gateway` showed `Degraded` because cert-manager was still completing the Let's Encrypt ACME HTTP-01 challenge for the `nip.io` domain. All application pods were healthy — this was a TLS timing issue only.

---

## Step 4 — Issue: StorageClass Conflict

The `gp3` StorageClass already existed in the cluster from a prior `terraform apply`. Kubernetes forbids updating StorageClass parameters, causing a `SyncFailed` error.

**Fix:** Delete the existing StorageClass so ArgoCD could recreate it cleanly from Git:

```bash
kubectl delete storageclass gp3
argocd app sync bankapp
# storageclass.storage.k8s.io/gp3 created — Synced ✅
```

---

## Step 5 — Self-Healing Tests

`selfHeal: true` means ArgoCD continuously reconciles the cluster against Git and corrects any drift.

### Test 1 — Manual Scale Down

```bash
kubectl scale deployment bankapp -n bankapp --replicas=1
```

**Result:** ArgoCD detected the drift within ~60 seconds and restored replicas to 4. ✅

### Test 2 — Delete a Service

```bash
kubectl delete service bankapp-service -n bankapp
```

**Result:** ArgoCD recreated `bankapp-service` in **9 seconds**. ✅

### Test 3 — Inject a Rogue Environment Variable

```bash
kubectl set env deployment/bankapp -n bankapp ROGUE_VAR=hacked
```

**Result:** ArgoCD detected `OutOfSync` and showed the drift via `argocd app diff`. However, `ROGUE_VAR` was not auto-reverted.

**Why?** With `ServerSideApply=true`, ArgoCD only manages fields it **owns**. The env var was injected by `kubectl` under a different field manager — ArgoCD does not overwrite fields owned by other managers. This is by design, not a bug.

**Cleanup:**
```bash
kubectl set env deployment/bankapp -n bankapp ROGUE_VAR-
```

### Self-Healing Summary

| Test | Expected | Result |
|------|----------|--------|
| Scale to 1 replica | Restored to 4 | ✅ Auto-healed in ~60s |
| Delete service | Recreated | ✅ Recreated in 9s |
| Inject env var | Reverted | ⚠️ Detected but not reverted (ServerSideApply field ownership) |

---

## Step 6 — GitOps Deployment Flow

This demonstrates the core GitOps loop: **commit to Git → ArgoCD detects → syncs to cluster**.

### Change: Scale Down Replicas via Git

```bash
# Edit the manifest
sed -i 's/  replicas: 4/  replicas: 2/' k8s/bankapp-deployment.yml

# Commit and push
git add k8s/bankapp-deployment.yml
git commit -m "scale: reduce bankapp replicas from 4 to 2"
git push origin feat/gitops
```

```bash
# Trigger sync (or wait up to 3 minutes for auto-detection)
argocd app sync bankapp

# Verify
kubectl get deployment bankapp -n bankapp -o jsonpath='{.spec.replicas}'
# 2 ✅
```

**ArgoCD picked up commit `6ba9c8a`** and applied the change. No `kubectl` was used to change the cluster — Git was the only interface.

### Deployment Audit Trail

```bash
argocd app history bankapp
```

```
SOURCE  https://github.com/shivkumarkonnuri/AI-BankApp-DevOps-EKS.git
ID   DATE                           REVISION
0    2026-05-23 12:27:35 +0000 UTC  feat/gitops (9f18632)
1    2026-05-23 12:42:48 +0000 UTC  feat/gitops (6ba9c8a)
```

Every deployment is traceable to an exact Git commit SHA — who changed what, when, and why (via commit message).

---

## Key Learnings

- **Git is the single source of truth.** The cluster is always a reflection of the Git repo.
- **`selfHeal: true`** continuously reconciles drift — deleted or modified resources are restored automatically.
- **`prune: true`** removes cluster resources when their manifests are deleted from Git.
- **`ServerSideApply=true`** means ArgoCD only manages the fields it owns. Other tools can coexist without conflict, but ArgoCD won't revert their changes.
- **Every deployment is a Git commit** — full audit trail, instant rollback via `git revert`.
- **ArgoCD eliminates the need for cluster credentials in CI pipelines** — only ArgoCD touches the cluster directly.

---

## Git Log

```
6ba9c8a  scale: reduce bankapp replicas from 4 to 2
9f18632  fix: use client-side apply for StorageClass to avoid parameter conflict
118a514  fix: skip StorageClass sync to avoid parameter update conflict
4ab6672  feat: add ArgoCD application manifest
```

---

*Day 84 — GitOps with ArgoCD on EKS | May 23, 2026*
