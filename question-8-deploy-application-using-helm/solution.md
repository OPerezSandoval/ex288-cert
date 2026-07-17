

## Solution (Timed Exercise - 25 minutes)

### Step 1: Switch to Project
```bash
oc project exam-resource
```

---

### Step 2: Navigate to Helm Chart Directory
```bash
cd /home/devop/exam-api
```

---

### Step 3: Set Chart Version in Chart.yaml
```bash
vi Chart.yaml
```

**Modify the version:**
```yaml
apiVersion: v2
name: exam-api
description: A Helm chart for exam-api application
type: application
version: 0.3.22        # Change this to 0.3.22
appVersion: "1.0.0"
```

Save with `:wq`

---

### Step 4: Update values.yaml
```bash
vi values.yaml
```

**Replace with corrected values:**
```yaml
# Values for exam-api
replicaCount: 2

image:
  repository: image-registry.openshift-image-registry.svc:5000/exam-resource/ex288-api
  pullPolicy: IfNotPresent
  tag: "latest"

imagePullSecrets: []
nameOverride: ""
fullnameOverride: ""

serviceAccount:
  create: true
  annotations: {}
  name: ""

podAnnotations: {}

podSecurityContext: {}

securityContext: {}

service:
  type: ClusterIP
  port: 8080

ingress:
  enabled: true
  className: ""
  annotations: {}
  hosts:
    - host: exam-api-exam-resource.apps.ocp4.example.com
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []

# Environment-specific settings
environment: prod

# Environment variables to inject into pods (will iterate with range)
env:
  - name: ENVIRONMENT
    value: "prod"
  - name: API_KEY
    value: "prod-secret-key-12345"
  - name: DB_HOST
    value: "postgres-prod.example.com"
  - name: APP_VERSION
    value: "1.0.0"

# Resource limits based on environment
resources:
  prod:
    limits:
      cpu: 500m
      memory: 512Mi
    requests:
      cpu: 250m
      memory: 256Mi
  dev:
    limits:
      cpu: 200m
      memory: 256Mi
    requests:
      cpu: 100m
      memory: 128Mi
  qa:
    limits:
      cpu: 300m
      memory: 384Mi
    requests:
      cpu: 150m
      memory: 192Mi

nodeSelector: {}

tolerations: []

affinity: {}
```

Save with `:wq`

---

### Step 5: Update templates/deployment.yaml

**Add range for environment variables and if statement for resources:**
```bash
vi templates/deployment.yaml
```

**Replace with this template:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "exam-api.fullname" . }}
  labels:
    {{- include "exam-api.labels" . | nindent 4 }}
    tag: {{ .Values.image.tag }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "exam-api.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "exam-api.selectorLabels" . | nindent 8 }}
        tag: {{ .Values.image.tag }}
    spec:
      serviceAccountName: {{ include "exam-api.serviceAccountName" . }}
      containers:
      - name: {{ .Chart.Name }}
        image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
        imagePullPolicy: {{ .Values.image.pullPolicy }}
        ports:
        - name: http
          containerPort: 8080
          protocol: TCP
        livenessProbe:
          httpGet:
            path: /q/health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
          timeoutSeconds: 5
        readinessProbe:
          httpGet:
            path: /q/health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
        env:
        {{- range .Values.env }}
        - name: {{ .name }}
          value: {{ .value | quote }}
        {{- end }}
        {{- if eq .Values.environment "prod" }}
        resources:
          {{- toYaml .Values.resources.prod | nindent 10 }}
        {{- else if eq .Values.environment "dev" }}
        resources:
          {{- toYaml .Values.resources.dev | nindent 10 }}
        {{- else if eq .Values.environment "qa" }}
        resources:
          {{- toYaml .Values.resources.qa | nindent 10 }}
        {{- end }}
```

**Key additions:**
1. **Range loop** for environment variables:
```yaml
   {{- range .Values.env }}
   - name: {{ .name }}
     value: {{ .value | quote }}
   {{- end }}
```

2. **If statement** for environment-based resources:
```yaml
   {{- if eq .Values.environment "prod" }}
   resources:
     {{- toYaml .Values.resources.prod | nindent 10 }}
   {{- else if eq .Values.environment "dev" }}
   resources:
     {{- toYaml .Values.resources.dev | nindent 10 }}
   {{- else if eq .Values.environment "qa" }}
   resources:
     {{- toYaml .Values.resources.qa | nindent 10 }}
   {{- end }}
```

Save with `:wq`

---

### Step 6: Update templates/service.yaml
```bash
vi templates/service.yaml
```

**Change port from 80 to 8080:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "exam-api.fullname" . }}
  labels:
    {{- include "exam-api.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: 8080
      protocol: TCP
      name: http
  selector:
    {{- include "exam-api.selectorLabels" . | nindent 4 }}
```

Save with `:wq`

---

### Step 7: Update templates/ingress.yaml for OpenShift Route

**OpenShift uses Routes, not Ingress. Replace ingress.yaml with route.yaml:**
```bash
rm templates/ingress.yaml

cat > templates/route.yaml <<'EOF'
{{- if .Values.ingress.enabled -}}
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: {{ include "exam-api.fullname" . }}
  labels:
    {{- include "exam-api.labels" . | nindent 4 }}
spec:
  host: {{ (index .Values.ingress.hosts 0).host }}
  to:
    kind: Service
    name: {{ include "exam-api.fullname" . }}
  port:
    targetPort: http
{{- end }}
EOF
```

---

### Step 8: Validate Helm Chart
```bash
# Lint the chart
helm lint .

# Test template rendering
helm template exam-api .
```

**Expected output:** No errors, templates render correctly with environment variables and conditional resources.

---

### Step 9: Install Helm Chart
```bash
helm install exam-api .
```

**Expected output:**
```
NAME: exam-api
LAST DEPLOYED: ...
NAMESPACE: exam-resource
STATUS: deployed
REVISION: 1
```

---

### Step 10: Verify Deployment
```bash
# Check all resources
helm list

# Check pods (should be 2 replicas)
oc get pods

# Check service
oc get svc

# Check route
oc get route
```

**Expected pods:**
```
NAME                        READY   STATUS    RESTARTS   AGE
exam-api-xxxxxxxxxx-xxxxx   1/1     Running   0          30s
exam-api-xxxxxxxxxx-xxxxx   1/1     Running   0          30s
```

---

### Step 11: Verify Environment Variables
```bash
# Check env vars in one of the pods
POD=$(oc get pods -l app.kubernetes.io/name=exam-api -o jsonpath='{.items[0].metadata.name}')

oc exec $POD -- env | grep -E 'ENVIRONMENT|API_KEY|DB_HOST|APP_VERSION'
```

**Expected output:**
```
ENVIRONMENT=prod
API_KEY=prod-secret-key-12345
DB_HOST=postgres-prod.example.com
APP_VERSION=1.0.0
```

---

### Step 12: Verify Resources Based on Environment
```bash
# Check resources applied
oc get deployment exam-api -o yaml | grep -A 10 resources
```

**Expected output (since environment=prod):**
```yaml
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi
```

---

### Step 13: Test Application
```bash
# Test health endpoint
curl http://exam-api-exam-resource.apps.ocp4.example.com/q/health
```

**Expected output:**
```json
{"status":"UP","environment":"prod"}
```

**Test main endpoint:**
```bash
curl http://exam-api-exam-resource.apps.ocp4.example.com/
```

**Expected output:**
```json
{
  "message": "Production API - High Performance Mode",
  "environment": "prod",
  "version": "1.0.0",
  "config": {
    "apiKey": "prod-secret-key-12345",
    "dbHost": "postgres-prod.example.com"
  }
}
```

---

### Step 14: Verify Chart Version
```bash
helm list
```

**Should show:**
```
NAME      NAMESPACE      REVISION  UPDATED                                 STATUS    CHART            APP VERSION
exam-api  exam-resource  1         2025-11-17 15:30:00.000000000 -0500 EST deployed  exam-api-0.3.22  1.0.0
```

---

### Step 15: Test Different Environment (Optional Verification)

**Update values to test dev environment:**
```bash
# Upgrade with dev environment
helm upgrade exam-api . --set environment=dev --set env[0].value=dev

# Check resources changed
oc get deployment exam-api -o yaml | grep -A 10 resources
```

**Should show dev resources:**
```yaml
resources:
  limits:
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi
```

**Test API response:**
```bash
curl http://exam-api-exam-resource.apps.ocp4.example.com/
```

**Should show:**
```json
{
  "message": "Development API - Debug Mode Enabled",
  "environment": "dev",
  ...
}
```

---

## Success Criteria

- Project `exam-resource` exists
- Application `exam-api` is deployed via Helm
- Chart version is `0.3.22`
- Application uses image `image-registry.openshift-image-registry.svc:5000/exam-resource/ex288-api:latest`
- Application has **2 replicas** running
- Label `tag: latest` is applied
- Route `exam-api-exam-resource.apps.ocp4.example.com` is accessible
- Liveness and readiness probes configured for `/q/health`
- Environment variables are injected using **range** loop
- Resources are conditionally set based on environment using **if statement**
- Application returns different responses based on `ENVIRONMENT` variable
- Health endpoint returns `{"status":"UP","environment":"prod"}`

---

## Key Commands Reference
```bash
# Create Helm chart
helm create <chart-name>

# Lint chart
helm lint <chart-path>

# Render templates (dry-run)
helm template <release-name> <chart-path>

# Install chart
helm install <release-name> <chart-path>

# Upgrade chart
helm upgrade <release-name> <chart-path>

# List releases
helm list

# Uninstall release
helm uninstall <release-name>

# Get values
helm get values <release-name>

# Rollback
helm rollback <release-name> <revision>

# Show chart info
helm show chart <chart-path>
helm show values <chart-path>
```

---

## Helm Template Functions Used

### Range (Loop)
```yaml
{{- range .Values.env }}
- name: {{ .name }}
  value: {{ .value | quote }}
{{- end }}
```

Iterates through a list in `values.yaml`.

### If/Else (Conditional)
```yaml
{{- if eq .Values.environment "prod" }}
resources:
  {{- toYaml .Values.resources.prod | nindent 10 }}
{{- else if eq .Values.environment "dev" }}
resources:
  {{- toYaml .Values.resources.dev | nindent 10 }}
{{- else if eq .Values.environment "qa" }}
resources:
  {{- toYaml .Values.resources.qa | nindent 10 }}
{{- end }}
```

Conditionally includes content based on values.

### Common Template Functions

| Function | Purpose | Example |
|----------|---------|---------|
| `{{ .Values.key }}` | Access value | `{{ .Values.image.tag }}` |
| `{{- if }}` | Conditional | `{{- if .Values.ingress.enabled }}` |
| `{{- range }}` | Loop | `{{- range .Values.env }}` |
| `{{ include }}` | Include named template | `{{ include "exam-api.labels" . }}` |
| `{{ toYaml }}` | Convert to YAML | `{{ toYaml .Values.resources }}` |
| `{{ nindent }}` | Indent | `{{ toYaml .Values.resources | nindent 10 }}` |
| `{{ quote }}` | Add quotes | `{{ .value | quote }}` |
| `{{ eq }}` | Equal comparison | `{{ eq .Values.env "prod" }}` |

---

## Common Issues and Troubleshooting

| Issue | Symptom | Fix |
|-------|---------|-----|
| **Chart version wrong** | `helm list` shows different version | Edit `Chart.yaml`, set `version: 0.3.22` |
| **Wrong image** | ImagePullBackOff | Verify image exists: `oc get is ex288-api` |
| **Only 1 replica** | `oc get pods` shows 1 pod | Set `replicaCount: 2` in values.yaml |
| **Env vars not set** | API returns default values | Check range loop in deployment.yaml |
| **Resources not applied** | Wrong CPU/memory limits | Verify if statement checks `.Values.environment` |
| **Route not created** | No route exists | Set `ingress.enabled: true` in values.yaml |
| **Health probe fails** | Pod restarts | Verify `/q/health` endpoint works: `curl ...` |
| **Helm lint fails** | Template errors | Check YAML indentation, especially in range/if blocks |

---

## Understanding Helm Chart Structure
```
exam-api/
├── Chart.yaml              # Chart metadata (name, version, description)
├── values.yaml            # Default configuration values
├── templates/             # Kubernetes manifests as templates
│   ├── deployment.yaml    # Deployment with range and if
│   ├── service.yaml       # Service definition
│   ├── route.yaml         # OpenShift Route
│   ├── _helpers.tpl       # Template helpers (reusable snippets)
│   └── NOTES.txt          # Post-install notes
└── charts/                # Dependent charts (if any)
```