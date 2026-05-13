# Day 78 — Introduction to Helm and Chart Basics

## Environment
- OS: Windows 11 with WSL2 (Ubuntu 24.04)
- Cluster: Kind (Kubernetes in Docker) — ai-bankapp-cluster
- Tools: kubectl v1.36.1 | kind v0.23.0 | helm v3.20.2

---

## Task 1: Helm Concepts (In My Own Words)

### What is Helm?
Helm is the package manager for Kubernetes — just like `apt` is for Ubuntu or `npm` is for Node.js.
Instead of managing 12 separate YAML files for one application, Helm bundles everything into a
single reusable, versioned unit called a **chart**. One chart, many environments.

### Core Concepts

**Chart**
A collection of files that describe a set of Kubernetes resources. For example, a MySQL chart
contains templates for a StatefulSet, Service, Secret, and PVC — all in one package.
Think of it like a `.deb` package for Kubernetes.

**Release**
A running instance of a chart in your cluster. You can install the same chart multiple times
with different release names:
- `helm install bankapp-mysql ./mysql-chart` → release: bankapp-mysql
- `helm install staging-mysql ./mysql-chart` → release: staging-mysql

Both are isolated — separate secrets, separate PVCs, separate services.

**Repository**
A place where charts are stored and shared — like DockerHub but for Helm charts.
Example: Bitnami repo at `https://charts.bitnami.com/bitnami` hosts hundreds of charts.

**Values**
Configuration that customizes a chart for each deployment. Defined in `values.yaml` and
overridden at install time via `--set` or `-f values-file.yaml`.
Example: same chart, different values for dev vs prod:
- dev: `resources.requests.memory: 256Mi`
- prod: `resources.requests.memory: 2Gi`

### version vs appVersion in Chart.yaml

    apiVersion: v2
    name: mysql-chart
    version: 0.1.0      # The chart's own version — increment when you change templates/values
    appVersion: "8.0"   # The version of the app inside (MySQL 8.0 in our case)

They are completely independent. You can fix a bug in your chart templates (bump `version`
to `0.1.1`) without touching MySQL at all. `appVersion` is informational metadata.

### Why Helm Over Raw Manifests?

The AI-BankApp has 12 raw YAML files in `k8s/`. Problems with this approach:
- To change image tag: manually edit `bankapp-deployment.yml`
- To switch environments: manually update ConfigMaps, Secrets, resource limits in every file
- Secrets are hardcoded base64 in `secrets.yml` (e.g., `VGVzdEAxMjM=`)
- No rollback — `kubectl apply` has no revision history
- For 3 environments (dev/staging/prod): maintain 3 copies of 12 files = 36 files to manage

Helm solves all of this:

| Problem                    | Raw YAML                        | Helm                                      |
|----------------------------|---------------------------------|-------------------------------------------|
| Multi-environment config   | 3 copies of 12 files            | 1 chart + 3 values files                  |
| Secrets management         | Hardcoded base64                | Generated and managed by Helm             |
| Rollback                   | Manual git revert               | `helm rollback <release> <revision>`      |
| Dependency management      | Manual ordering                 | Chart dependencies in `Chart.yaml`        |
| Audit trail                | None                            | Full revision history via `helm history`  |

---

## Task 2: Cluster Setup

### Kind Cluster Config (`setup-k8s/kind-config.yml`)

    kind: Cluster
    apiVersion: kind.x-k8s.io/v1alpha4
    name: ai-bankapp-cluster
    nodes:
    - role: control-plane
      image: kindest/node:v1.35.0
      extraPortMappings:
        - containerPort: 30080
          hostPort: 8080
          protocol: TCP
    - role: worker
      image: kindest/node:v1.35.0
    - role: worker
      image: kindest/node:v1.35.0

`extraPortMappings` maps containerPort 30080 (inside the Kind Docker container) to hostPort
8080 on the Windows/WSL2 host. Kind nodes are Docker containers — without this mapping, a
NodePort service on 30080 is unreachable from your browser.

    kind create cluster --config setup-k8s/kind-config.yml
    kubectl get nodes

Output:

    NAME                               STATUS   ROLES           AGE   VERSION
    ai-bankapp-cluster-control-plane   Ready    control-plane   82s   v1.35.0
    ai-bankapp-cluster-worker          Ready    <none>          71s   v1.35.0
    ai-bankapp-cluster-worker2         Ready    <none>          71s   v1.35.0

### Raw Manifests in k8s/

    bankapp-deployment.yml  configmap.yml  hpa.yml               namespace.yml
    cert-manager.yml        gateway.yml    mysql-deployment.yml  ollama-deployment.yml
    pv.yml  pvc.yml  secrets.yml  service.yml

12 files — all hardcoded values. MySQL alone spans:
`mysql-deployment.yml` + `secrets.yml` + `pvc.yml` + `service.yml` + `configmap.yml`

---

## Task 3: Deploy MySQL Using a Custom Helm Chart

### Why a Custom Chart?
Bitnami paywalled their images in August 2025 — `docker.io/bitnami/mysql:*` images return
`not found` on Docker Hub. This is a real-world lesson: always have a fallback strategy.
Solution: built a minimal custom chart using the official `mysql:8.0` image.

### Chart Structure Created

    mysql-chart/
      Chart.yaml          # Chart metadata
      values.yaml         # Default configuration values
      templates/
        _helpers.tpl      # Reusable template helpers (scaffold default)
        secret.yaml       # Kubernetes Secret for DB credentials
        pvc.yaml          # PersistentVolumeClaim for MySQL data
        statefulset.yaml  # StatefulSet (not Deployment — see note below)
        service.yaml      # ClusterIP Service on port 3306

### Why StatefulSet and Not Deployment?
- **Deployment** pods are stateless and interchangeable. If a pod dies, it gets a fresh identity.
- **StatefulSet** pods have stable, persistent identities (`bankapp-mysql-0`) and their PVC
  stays attached even after restart.
- MySQL must reconnect to the **same** data volume after restart. A Deployment risks data loss
  because a restarted pod might mount a fresh empty PVC.

### Chart.yaml

    apiVersion: v2
    name: mysql-chart
    description: A Helm chart for MySQL - AI-BankApp
    type: application
    version: 0.1.0
    appVersion: "8.0"

### values.yaml

    image:
      repository: mysql      # Official Docker Hub MySQL image (no paywall)
      tag: "8.0"             # MySQL version
      pullPolicy: IfNotPresent

    auth:
      rootPassword: Test@123  # Root password — passed into Secret template
      database: bankappdb     # Database created on first startup

    service:
      type: ClusterIP          # Internal only — MySQL not exposed externally
      port: 3306               # Standard MySQL port

    resources:
      requests:
        memory: 256Mi          # Minimum guaranteed memory
        cpu: 250m              # 0.25 CPU cores guaranteed
      limits:
        memory: 512Mi          # Maximum memory before OOM kill
        cpu: 500m              # Maximum CPU — 0.5 cores

    storage:
      size: 5Gi                # PVC size for MySQL data directory

### Key Template — secret.yaml

`stringData` is used instead of `data` — Helm handles base64 encoding automatically:

    apiVersion: v1
    kind: Secret
    metadata:
      name: {{ .Release.Name }}-secret
    type: Opaque
    stringData:
      MYSQL_ROOT_PASSWORD: {{ .Values.auth.rootPassword }}
      MYSQL_DATABASE: {{ .Values.auth.database }}

**Go Template syntax explained:**
- `{{ .Release.Name }}` — substituted with the release name at install time (`bankapp-mysql`)
- `{{ .Values.auth.rootPassword }}` — pulled from `values.yaml` (or `--set` override)
- Installing the same chart twice with different release names creates completely isolated
  resources — no naming conflicts between `bankapp-mysql-secret` and `staging-mysql-secret`

### Dry Run Before Install

    helm template bankapp-mysql ~/AI-BankApp-DevOps-EKS/mysql-chart
    helm lint ~/AI-BankApp-DevOps-EKS/mysql-chart

`helm template` renders all templates locally with values substituted — without touching the
cluster. Essential for debugging. Output confirmed:
- `image: "mysql:8.0"` ✅
- `name: bankapp-mysql-secret` ✅
- Resources correctly substituted ✅
- `1 chart(s) linted, 0 chart(s) failed` ✅

### Install

    helm install bankapp-mysql ~/AI-BankApp-DevOps-EKS/mysql-chart

Output:

    NAME: bankapp-mysql
    NAMESPACE: default
    STATUS: deployed
    REVISION: 1

Pod came up in 7 seconds — `1/1 Running`.

### Verify

    kubectl get all,pvc,secret | grep bankapp

Output:

    pod/bankapp-mysql-0                              1/1     Running
    service/bankapp-mysql                            ClusterIP   10.96.203.244   3306/TCP
    statefulset.apps/bankapp-mysql                   1/1
    persistentvolumeclaim/bankapp-mysql-pvc          Bound    5Gi
    secret/bankapp-mysql-secret                      Opaque   2
    secret/sh.helm.release.v1.bankapp-mysql.v1       helm.sh/release.v1   1

Note: `sh.helm.release.v1.bankapp-mysql.v1` — Helm stores release state as a Kubernetes
Secret. This is how `helm rollback` works — it reads old revision secrets and re-applies
that configuration. No external database needed.

    kubectl exec -it bankapp-mysql-0 -- mysql -uroot -pTest@123 -e "SHOW DATABASES;"

Output:

    +--------------------+
    | Database           |
    +--------------------+
    | bankappdb          |   <- our database created successfully
    | information_schema |
    | mysql              |
    | performance_schema |
    | sys                |
    +--------------------+

---

## Task 4: Values File

`--set` gets unwieldy fast — 10+ flags on the command line is error-prone.
Values files (`-f`) are the preferred approach for anything beyond 1-2 overrides.

### mysql-values.yaml

    image:
      repository: mysql
      tag: "8.0"
      pullPolicy: IfNotPresent

    auth:
      rootPassword: Test@123
      database: bankappdb

    service:
      type: ClusterIP
      port: 3306

    resources:
      requests:
        memory: 256Mi
        cpu: 250m
      limits:
        memory: 512Mi
        cpu: 500m

    storage:
      size: 5Gi

Upgrade using values file:

    helm upgrade bankapp-mysql ~/AI-BankApp-DevOps-EKS/mysql-chart \
      -f ~/AI-BankApp-DevOps-EKS/mysql-values.yaml

---

## Task 5: Release Management — Upgrade, Rollback, Uninstall

### helm history

Every change is tracked as a revision:

    helm history bankapp-mysql

Output:

    REVISION  UPDATED                         STATUS      CHART              DESCRIPTION
    1         Wed May 13 10:32:28 2026        superseded  mysql-chart-0.1.0  Install complete
    2         Wed May 13 10:34:54 2026        superseded  mysql-chart-0.1.0  Upgrade complete
    3         Wed May 13 10:35:31 2026        deployed    mysql-chart-0.1.0  Rollback to 1

### Rollback

    helm rollback bankapp-mysql 1

Key insight: Helm **never rewrites history**. A rollback creates a new revision (3) that
applies revision 1's configuration. The full audit trail is always preserved — you can see
exactly what changed and when. With `kubectl apply` there is no built-in rollback — you'd
have to `git revert` and re-apply manually.

### helm list output

    NAME           NAMESPACE  REVISION  STATUS    CHART              APP VERSION
    bankapp-mysql  default    3         deployed  mysql-chart-0.1.0  8.0

---

## Task 6: Exploring a Production Chart Structure

    helm pull bitnami/mysql --untar
    ls mysql/

Output:

    Chart.lock  Chart.yaml  README.md  charts/  templates/  values.schema.json  values.yaml

    ls mysql/templates/

Output:

    NOTES.txt     cert.yaml        networkpolicy.yaml  role.yaml         secrets.yaml
    _helpers.tpl  extra-list.yaml  primary/            rolebinding.yaml  serviceaccount.yaml
    ca-cert.yaml  metrics-svc.yaml prometheusrule.yaml secondary/        servicemonitor.yaml

    ls mysql/templates/primary/

Output:

    configmap.yaml  initialization-configmap.yaml  pdb.yaml  startdb-configmap.yaml
    statefulset.yaml  svc-headless.yaml  svc.yaml

### Comparison: Our Chart vs Bitnami Production Chart

| Aspect           | Our Custom Chart       | Bitnami MySQL Chart                              |
|------------------|------------------------|--------------------------------------------------|
| Template files   | 4                      | 15+                                              |
| Template dirs    | flat                   | `primary/`, `secondary/`                         |
| Dependencies     | none                   | `common` library chart                           |
| Extra files      | —                      | `values.schema.json`, `Chart.lock`, `README.md`  |
| Features         | basic MySQL            | TLS, metrics, replication, network policies      |
| Images           | official `mysql:8.0`   | Bitnami (paywalled Aug 2025)                     |

### Key Files Explained

**Chart.lock** — locks exact dependency versions (like `package-lock.json` in Node.js).
Ensures everyone gets the exact same `common` library version, not just `2.x.x`.

**values.schema.json** — validates values passed to the chart. If you pass `replicas: "three"`
instead of `replicas: 3`, Helm rejects it immediately with a clear error before anything deploys.

**_helpers.tpl** — reusable template snippets (like functions). Instead of repeating
`{{ .Release.Name }}-mysql` in every template, define it once as `{{ include "mysql.fullname" . }}`
and reuse everywhere. Keeps templates DRY.

**NOTES.txt** — the post-install message printed to terminal after `helm install`. The
connection instructions you see after deploying are rendered from this file.

**charts/** — downloaded subchart dependencies live here (populated by `helm dependency update`).

---

## Real-World Lesson: Bitnami Paywall (Aug 2025)

During this task, Bitnami chart installs failed with `ImagePullBackOff`:

    Failed to pull image "docker.io/bitnami/mysql:9.4.0-debian-12-r1": not found

Bitnami paywalled all images on Docker Hub from August 28, 2025. Even pinning to older chart
versions (9.x, 12.x) failed because all Bitnami image tags were removed.

Debugging steps taken:
1. `kubectl describe pod bankapp-mysql-0` → identified `ImagePullBackOff` on init container
2. Tried `--set image.repository=mysql` override → chart init container still used Bitnami image
3. Tried `global.security.allowInsecureImages=true` → init container incompatible with official image
4. Tried older chart versions → same paywall issue on all tags
5. **Solution:** Built a custom chart from scratch using `helm create` with official `mysql:8.0`

Takeaway: In production, teams maintain their own chart repos or use enterprise registries
rather than depending on public chart repos that can change their policies.

---

## Why the AI-BankApp's 12 Raw YAML Files Benefit from Helm

| Current Pain                                              | Helm Solution                                          |
|-----------------------------------------------------------|--------------------------------------------------------|
| `secrets.yml` has hardcoded base64 passwords             | Secret template with `{{ .Values.auth.rootPassword }}` |
| `pvc.yml` hardcodes `storageClassName: gp3` (AWS-only)  | `storage.storageClass: ""` — empty = cluster default   |
| Image tags hardcoded in every deployment file            | `image.tag` value — change once, applies everywhere    |
| 12 separate files to apply in correct order              | `helm install` handles ordering and dependencies       |
| No rollback after `kubectl apply`                        | `helm rollback <release> <revision>`                   |
| Duplicating all 12 files for dev/staging/prod            | One chart + `dev-values.yaml`, `prod-values.yaml`      |

---

## Commands Reference

    # Repository management
    helm repo add <name> <url>
    helm repo update
    helm search repo <chart> --versions

    # Release management
    helm install <release> <chart> [--set key=val] [-f values.yaml]
    helm upgrade <release> <chart> [-f values.yaml]
    helm upgrade --install <release> <chart>   # install if missing, upgrade if exists
    helm rollback <release> <revision>
    helm uninstall <release>

    # Inspection
    helm list [-A]                    # list releases (all namespaces with -A)
    helm history <release>            # revision history
    helm status <release>             # current status
    helm show values <chart>          # all configurable values

    # Debugging
    helm template <release> <chart>   # render templates locally without deploying
    helm lint <chart>                 # validate chart structure
    helm pull <chart> --untar         # download chart locally to inspect
