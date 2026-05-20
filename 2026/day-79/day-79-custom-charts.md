# Day 79 — Creating a Custom Helm Chart for AI-BankApp

## Overview

Converted 12 raw Kubernetes manifests from the `k8s/` directory into a single, production-grade Helm chart for the AI-BankApp stack — a Spring Boot banking application with MySQL and an Ollama AI chatbot (TinyLlama).

**Result:** The entire 3-tier stack deploys with one command. One boolean disables the AI chatbot entirely.

```bash
helm install my-bankapp bankapp/ \
  -n bankapp --create-namespace \
  --set storageClass.create=false \
  --set mysql.persistence.storageClass=standard \
  --set ollama.persistence.storageClass=standard
```

---

## Chart Structure

```
helm-chart/
└── bankapp/
    ├── Chart.yaml
    ├── values.yaml
    └── templates/
        ├── _helpers.tpl
        ├── NOTES.txt
        ├── configmap.yaml
        ├── secrets.yaml
        ├── storage.yaml
        ├── services.yaml
        ├── bankapp-deployment.yaml
        ├── mysql-deployment.yaml
        ├── ollama-deployment.yaml
        └── hpa.yaml
```

---

## Raw Manifests → Helm Templates (Side-by-Side)

### 1. Secrets — eliminating manual base64 encoding

**Before (`k8s/secrets.yml`):**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: bankapp-secret
type: Opaque
data:
  MYSQL_ROOT_PASSWORD: VGVzdEAxMjM=   # manually base64 encoded
  MYSQL_USER: cm9vdA==
  MYSQL_PASSWORD: VGVzdEAxMjM=
```

**After (`templates/secrets.yaml`):**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: {{ include "bankapp.fullname" . }}-secret
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "bankapp.labels" . | nindent 4 }}
type: Opaque
data:
  MYSQL_ROOT_PASSWORD: {{ .Values.secrets.mysqlRootPassword | b64enc | quote }}
  MYSQL_USER: {{ .Values.secrets.mysqlUser | b64enc | quote }}
  MYSQL_PASSWORD: {{ .Values.secrets.mysqlPassword | b64enc | quote }}
```

`b64enc` handles encoding automatically. Credentials live in `values.yaml` as plain text and can be overridden per environment at install time with `--set`.

---

### 2. Ollama Deployment — model name as a configurable value

**Before (`k8s/ollama-deployment.yml`):**
```yaml
lifecycle:
  postStart:
    exec:
      command:
        - /bin/sh
        - -c
        - |
          until ollama list > /dev/null 2>&1; do sleep 2; done
          ollama pull tinyllama    # hardcoded model
readinessProbe:
  exec:
    command: ["/bin/sh", "-c", "ollama list | grep -q tinyllama"]  # hardcoded
```

**After (`templates/ollama-deployment.yaml`):**
```yaml
lifecycle:
  postStart:
    exec:
      command:
        - /bin/sh
        - -c
        - |
          until ollama list > /dev/null 2>&1; do sleep 2; done
          ollama pull {{ .Values.ollama.model }}    # configurable
readinessProbe:
  exec:
    command: ["/bin/sh", "-c", "ollama list | grep -q {{ .Values.ollama.model }}"]
```

Switch from TinyLlama to Llama3 with `--set ollama.model=llama3` — no YAML editing.

---

### 3. BankApp Deployment — conditional init containers

**After (`templates/bankapp-deployment.yaml`):**
```yaml
initContainers:
  - name: wait-for-mysql
    image: busybox:1.36
    command: ["/bin/sh", "-c", "until nc -z {{ include "bankapp.fullname" . }}-mysql 3306; do sleep 2; done"]
  {{- if .Values.ollama.enabled }}
  - name: wait-for-ollama
    image: busybox:1.36
    command: ["/bin/sh", "-c", "until nc -z {{ include "bankapp.fullname" . }}-ollama 11434; do sleep 2; done"]
  {{- end }}
```

When `ollama.enabled=false`, the init container is removed automatically. The app won't wait for a service that doesn't exist.

---

## values.yaml

```yaml
# BankApp configuration
bankapp:
  replicaCount: 4
  image:
    repository: shivkumarkonnuri/ai-bankapp-eks
    tag: latest
    pullPolicy: Always
  resources:
    requests:
      memory: "256Mi"
      cpu: "250m"
    limits:
      memory: "512Mi"
      cpu: "500m"
  service:
    type: ClusterIP
    port: 8080
  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 4
    targetCPUUtilization: 70

# MySQL configuration
mysql:
  enabled: true
  image:
    repository: mysql
    tag: "8.0"
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

# Ollama AI configuration
ollama:
  enabled: true
  image:
    repository: ollama/ollama
    tag: "latest"
  model: tinyllama          # switch model here — no template changes needed
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

# Shared configuration
config:
  mysqlDatabase: bankappdb
  ollamaUrl: ""             # auto-generated from service name if empty

# Secrets (plain text — b64enc applied in template)
secrets:
  mysqlRootPassword: Test@123
  mysqlUser: root
  mysqlPassword: Test@123

# StorageClass
storageClass:
  create: true
  name: gp3
  provisioner: ebs.csi.aws.com

# Gateway (optional — for EKS with Envoy Gateway)
gateway:
  enabled: false
  hostname: ""
  tls:
    enabled: false
```

---

## Go Template Syntax Cheat Sheet

| Syntax | What it does |
|--------|-------------|
| `{{ .Values.bankapp.replicaCount }}` | Reads a value from values.yaml |
| `{{ .Release.Name }}` | Release name passed at install |
| `{{ .Release.Namespace }}` | Namespace of the release |
| `{{ include "bankapp.fullname" . }}` | Calls a named helper from `_helpers.tpl` (pipeable) |
| `{{ template "bankapp.labels" . }}` | Calls a helper (not pipeable) |
| `{{- if .Values.ollama.enabled }}` | Conditional block; `{{-` trims leading whitespace |
| `{{- end }}` | Closes a block |
| `{{ .Values.secrets.mysqlRootPassword \| b64enc \| quote }}` | Pipes: base64 encode then wrap in quotes |
| `{{ .Values.config.mysqlDatabase \| quote }}` | Wraps value in double quotes |
| `{{- toYaml . \| nindent 12 }}` | Converts object to YAML and indents 12 spaces |
| `{{ default "fallback" .Values.someKey }}` | Uses fallback if value is empty |
| `{{ printf "http://%s-ollama:11434" (include "bankapp.fullname" .) }}` | String formatting |

---

## Debugging Workflow Discovered

The most important lesson from today: **`helm lint` is not enough.**

```bash
# ❌ This passes even for structurally broken charts
helm lint bankapp/

# ✅ This validates against the actual Kubernetes API schema
helm template my-bankapp bankapp/ | kubectl apply --dry-run=client -f -
```

### Bugs caught only by `--dry-run=client`:

| Bug | Symptom | Silent? |
|-----|---------|---------|
| `spec:` under `metadata:` | Entire Deployment spec ignored | ✅ lint passes |
| `strategy:` + `template:` under `selector:` | Containers unreachable | ✅ lint passes |
| `containerPorts:` instead of `containerPort:` | Port never exposed | ✅ lint passes |
| `mountpath:` instead of `mountPath:` | Volume never mounted | ✅ lint passes |
| HPA `spec:` under `metadata:` | maxReplicas=0, scaleTargetRef empty | ✅ lint passes |

### Full debug toolkit:

```bash
# Render all templates to stdout
helm template my-bankapp bankapp/

# Check what resource kinds are rendered
helm template my-bankapp bankapp/ | grep "^kind:"

# Inspect specific resource with context
helm template my-bankapp bankapp/ --debug 2>&1 | grep -A 30 "kind: Deployment"

# Test conditional rendering (ollama disabled)
helm template my-bankapp bankapp/ --set ollama.enabled=false | grep "^kind:"

# Test value overrides
helm template my-bankapp bankapp/ \
  --set bankapp.image.tag=abc1234 \
  --set bankapp.autoscaling.enabled=false \
  --set bankapp.replicaCount=2 | grep -E "image:|replicas:"

# Kubernetes API schema validation (THE IMPORTANT ONE)
helm template my-bankapp bankapp/ | kubectl apply --dry-run=client -f -

# Full dry-run with Helm debug output
helm install my-bankapp bankapp/ --dry-run --debug -n bankapp --create-namespace
```

---

## helm template Output (rendered manifests)

```
kind: Secret            → my-bankapp-secret
kind: ConfigMap         → my-bankapp-config
kind: StorageClass      → gp3  (skipped with storageClass.create=false)
kind: PersistentVolumeClaim → my-bankapp-mysql-pvc (5Gi)
kind: PersistentVolumeClaim → my-bankapp-ollama-pvc (10Gi)
kind: Service           → my-bankapp-mysql:3306
kind: Service           → my-bankapp-ollama:11434
kind: Service           → my-bankapp-service:8080
kind: Deployment        → my-bankapp (Spring Boot, HPA-managed)
kind: Deployment        → my-bankapp-mysql
kind: Deployment        → my-bankapp-ollama
kind: HorizontalPodAutoscaler → my-bankapp-hpa (2-4 replicas, 70% CPU)
```

All 11 objects validated:
```
secret/my-bankapp-secret created (dry run)
configmap/my-bankapp-config created (dry run)
persistentvolumeclaim/my-bankapp-mysql-pvc created (dry run)
persistentvolumeclaim/my-bankapp-ollama-pvc created (dry run)
service/my-bankapp-mysql created (dry run)
service/my-bankapp-ollama created (dry run)
service/my-bankapp-service created (dry run)
deployment.apps/my-bankapp created (dry run)
deployment.apps/my-bankapp-mysql created (dry run)
deployment.apps/my-bankapp-ollama created (dry run)
horizontalpodautoscaler.autoscaling/my-bankapp-hpa created (dry run)
```

---

## Deployment Verification

```bash
$ kubectl get pods -n bankapp
NAME                                 READY   STATUS    RESTARTS   AGE
my-bankapp-6cc5c94888-n6mvv          1/1     Running   0          128m
my-bankapp-6cc5c94888-tcsqh          1/1     Running   0          128m
my-bankapp-mysql-76f49c8554-rhkt5    1/1     Running   0          128m
my-bankapp-ollama-55bb44df54-p7fwq   1/1     Running   0          128m

$ kubectl get hpa -n bankapp
NAME             REFERENCE               TARGETS              MINPODS   MAXPODS   REPLICAS   AGE
my-bankapp-hpa   Deployment/my-bankapp   cpu: <unknown>/70%   2         4         2          128m

$ curl http://localhost:8080/actuator/health
{"status":"UP","groups":["liveness","readiness"]}
```

---

## Effect of `ollama.enabled=false`

```bash
$ helm template my-bankapp bankapp/ --set ollama.enabled=false | grep "^kind:"
kind: Secret
kind: ConfigMap
kind: StorageClass
kind: PersistentVolumeClaim      # only mysql-pvc, ollama-pvc removed
kind: Service                    # mysql service
kind: Service                    # bankapp service (ollama service removed)
kind: Deployment                 # bankapp (ollama init container removed)
kind: Deployment                 # mysql
kind: HorizontalPodAutoscaler
```

One boolean removes: Ollama Deployment, Ollama Service, Ollama PVC, and the `wait-for-ollama` init container from BankApp. Zero manual edits.

---

## Key Takeaways

1. `helm lint` checks Go template syntax only — it does NOT validate Kubernetes schema
2. Always pair with `helm template | kubectl apply --dry-run=client -f -`
3. YAML indentation is the #1 source of Helm bugs — `spec:` must always be a sibling of `metadata:`, never a child
4. `b64enc` eliminates manual base64 encoding of secrets
5. `{{- if }}` guards make components fully optional — one boolean controls an entire tier
6. Init containers + conditional logic ensure correct startup ordering regardless of which components are enabled
7. Ghost releases from failed installs must be cleaned with `helm uninstall` before retrying

---

## Startup Sequence

```
mysql pod        → ContainerCreating → Running → Ready (2m51s)
ollama pod       → Pulling image (large) → Running → postStart pulls tinyllama → Ready (~25m)
bankapp pod      → Init:0/2 → Init:1/2 (mysql ready) → Init:2/2 (ollama ready) → Running → Ready
```

The init containers enforce this ordering automatically — bankapp will not start until both dependencies are healthy.

---

## References

- Source repo: https://github.com/TrainWithShubham/AI-BankApp-DevOps (branch: `feat/gitops`)
- Helm docs: https://helm.sh/docs/chart_template_guide/
- My fork: https://github.com/shivkumarkonnuri/AI-BankApp-DevOps-EKS
