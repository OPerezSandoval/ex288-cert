# Question 8: Deploy an Application Using Helm

## Question

Deploy an application using Helm that meets the following requirements:

- The application uses the image: `registry.ocp4.example.com/ex288-api:latest`
- The application is part of a project named: **exam-resource**
- The application is named: **exam-api**
- The application is available at `http://exam-api-exam-resource.apps.ocp4.example.com`
- The application is deployed on **2 nodes** (replicas)
- The application configures **livenessProbe** and **readinessProbe** for `/q/health`
- The label **tag** is set to **latest**
- The chart version is **0.3.22**
- The project chart is located at `/home/devop/exam-api`
- The Helm chart uses **range** to iterate through environment variables from `values.yaml`
- The Helm chart uses **if statement** to conditionally set resources based on environment (prod/dev/qa)

**Note:** The application image runs at port **8080/TCP**

---

## Environment Setup

### Step 1: Create Project and Directory Structure
```bash
# Create project
oc new-project exam-resource
```

---

### Step 2: Create the API Application Code

**Create application directory:**
```bash
mkdir -p ~/question8/exam-api-source
cd ~/question8/exam-api-source
```

**Create a simple Node.js API:**
```bash
cat > app.js <<'EOF'
const http = require('http');

// Read environment variables
const environment = process.env.ENVIRONMENT || 'unknown';
const apiKey = process.env.API_KEY || 'not-set';
const dbHost = process.env.DB_HOST || 'not-set';
const appVersion = process.env.APP_VERSION || '1.0.0';

const hostname = '0.0.0.0';
const port = 8080;

const server = http.createServer((req, res) => {
  if (req.url === '/q/health') {
    // Health endpoint for probes
    res.statusCode = 200;
    res.setHeader('Content-Type', 'application/json');
    res.end(JSON.stringify({
      status: 'UP',
      environment: environment
    }));
  } else {
    // Main endpoint - returns different response based on environment
    res.statusCode = 200;
    res.setHeader('Content-Type', 'application/json');
    
    let message = '';
    if (environment === 'prod') {
      message = 'Production API - High Performance Mode';
    } else if (environment === 'dev') {
      message = 'Development API - Debug Mode Enabled';
    } else if (environment === 'qa') {
      message = 'QA API - Testing Environment';
    } else {
      message = 'API - Environment Not Specified';
    }
    
    res.end(JSON.stringify({
      message: message,
      environment: environment,
      version: appVersion,
      config: {
        apiKey: apiKey,
        dbHost: dbHost
      }
    }));
  }
});

server.listen(port, hostname, () => {
  console.log(`Server running at http://${hostname}:${port}/`);
  console.log(`Environment: ${environment}`);
});
EOF
```

**Create package.json:**
```bash
cat > package.json <<'EOF'
{
  "name": "exam-api",
  "version": "1.0.0",
  "description": "Simple API for EX288 Helm exercise",
  "main": "app.js",
  "scripts": {
    "start": "node app.js"
  },
  "keywords": ["api", "exam"],
  "author": "Student",
  "license": "MIT"
}
EOF
```

**Create Dockerfile:**
```bash
cat > Dockerfile <<'EOF'
FROM registry.access.redhat.com/ubi8/nodejs-16:latest

USER root

# Copy application files
COPY app.js package.json ./

# Install dependencies (none needed for this simple app)
RUN npm install --production

# Set permissions
RUN chown -R 1001:0 /opt/app-root/src && \
    chmod -R g+rwX /opt/app-root/src

USER 1001

EXPOSE 8080

CMD ["npm", "start"]
EOF
```

---

### Step 3: Build and Push Image to Internal Registry

**Build the container image:**
```bash
cd ~/exam-api-source

# Build with Podman
podman build -t ex288-api:latest .
```

**Tag for internal registry:**
```bash
# Get the internal registry route
REGISTRY=$(oc get route default-route -n openshift-image-registry -o jsonpath='{.spec.host}')

# Tag the image
podman tag ex288-api:latest ${REGISTRY}/exam-resource/ex288-api:latest
```

**Login to internal registry:**
```bash
# Get token
TOKEN=$(oc whoami -t)

# Login
podman login -u developer -p ${TOKEN} ${REGISTRY} --tls-verify=false
```

**Push image:**
```bash
podman push ${REGISTRY}/exam-resource/ex288-api:latest --tls-verify=false
```

**Verify image in registry:**
```bash
# Create imagestream if it doesn't exist
oc create imagestream ex288-api -n exam-resource

# Import the image
oc import-image ex288-api:latest \
  --from=${REGISTRY}/exam-resource/ex288-api:latest \
  --confirm \
  -n exam-resource

# Verify
oc get is ex288-api -n exam-resource
```

---

### Step 4: Create Base Helm Chart Structure
```bash
# Navigate to /home/devop
cd /question8/

# Create Helm chart
helm create exam-api

# Verify structure
ls -la exam-api/
```

**Expected structure:**
```
exam-api/
├── Chart.yaml
├── values.yaml
├── templates/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── _helpers.tpl
│   ├── NOTES.txt
│   └── tests/
└── charts/
```

---

### Step 5: Prepare values.yaml with Default (Incorrect) Values


**Create initial values.yaml (will need to modify this):**
```bash
cd /home/devop/exam-api

# You really only have to add the env varibles at the end to the current values.yaml
cat > values.yaml <<'EOF'
# Default values for exam-api
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

replicaCount: 1

image:
  repository: nginx
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
  port: 80

ingress:
  enabled: false
  className: ""
  annotations: {}
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []

resources: {}

nodeSelector: {}

tolerations: []

affinity: {}

# Environment-specific settings
environment: dev

# Environment variables to inject into pods
env:
  - name: ENVIRONMENT
    value: "dev"
  - name: API_KEY
    value: "default-key"
  - name: DB_HOST
    value: "localhost"
  - name: APP_VERSION
    value: "1.0.0"
EOF
```