## Solution (Timed Exercise - 10 minutes)

### Step 1: Create Project
```bash
oc new-project acid
```

---

### Step 2: Deploy Application from Container Image
```bash
oc new-app --name=phosphoric \
  --image=openshift/hello-openshift:latest
```

---

### Step 3: Wait for Initial Deployment
```bash
oc get pods
```

Wait until `phosphoric-xxxxx` shows `Running`.

**Test before ConfigMap (optional):**
```bash
# Expose temporarily to test
oc expose svc/phosphoric
oc get route

# Test default response
curl http://phosphoric-acid.apps.ocp4.example.com
```

**Expected output (without ConfigMap):**
```
Hello OpenShift!
```

---

### Step 4: Create ConfigMap
```bash
oc create configmap sedicen \
  --from-literal=RESPONSE="If you can see this your configmap works"
```

**Verify ConfigMap:**
```bash
oc get configmap sedicen
oc get configmap sedicen -o yaml
```

**Expected output:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: sedicen
  namespace: acid
data:
  RESPONSE: "If you can see this your configmap works"
```

---

### Step 5: Inject ConfigMap into Deployment
```bash
oc set env deployment/phosphoric --from=configmap/sedicen
```

**This command:**
- Reads all keys from the `sedicen` ConfigMap
- Sets them as environment variables in the `phosphoric` deployment
- Triggers a new rollout automatically

---

### Step 6: Wait for Rollout to Complete
```bash
oc rollout status deployment/phosphoric
```

**Expected output:**
```
deployment "phosphoric" successfully rolled out
```

**Verify pods are running:**
```bash
oc get pods
```

---

### Step 7: Verify Environment Variable (Optional)
```bash
# List environment variables set on deployment
oc set env deployment/phosphoric --list
```

**Expected output:**
```
# deployments/phosphoric, container phosphoric
RESPONSE=If you can see this your configmap works 
```

**Or check inside the pod:**
```bash
# Get pod name
POD=$(oc get pods -l deployment=phosphoric -o jsonpath='{.items[0].metadata.name}')

# Check environment variable
oc rsh $POD env | grep RESPONSE
```

**Expected output:**

RESPONSE=If you can see this your configmap works 

---

### Step 8: Expose Service
```bash
oc expose svc/phosphoric 
```

---

### Step 9: Verify Route
```bash
oc get route
```

**Expected output:**
```
NAME         HOST/PORT                                    PATH   SERVICES     PORT       TERMINATION   WILDCARD
phosphoric   phosphoric-acid.apps.ocp4.example.com               phosphoric   8080-tcp                 None
```

---

### Step 10: Test Application
```bash
curl http://phosphoric-acid.apps.ocp4.example.com
```

**Expected output:**
```
If you can see this your configmap works 
```

---

## Success Criteria

- Project `acid` exists
- Application `phosphoric` is running
- ConfigMap `sedicen` exists with key `RESPONSE` set to "Soda pop won't stop can't stop"
- Route `phosphoric-acid.apps.ocp4.example.com` is accessible
- Application returns: "Soda pop won't stop can't stop"
- Environment variable `RESPONSE` is set in the pod
- Updating ConfigMap and redeploying reflects new values

---

## Key Commands Reference
```bash
# Create project
oc new-project <name>

# Deploy from container image
oc new-app --name=<app> --image=<image>

# Create ConfigMap from literal
oc create configmap <name> --from-literal=<KEY>=<value>

# Create ConfigMap from file
oc create configmap <name> --from-file=<KEY>=<file>

# Inject ConfigMap into deployment
oc set env deployment/<app> --from=configmap/<configmap>

# List environment variables
oc set env deployment/<app> --list

# Expose service
oc expose svc/<app> --hostname=<hostname>

# Restart deployment (force rollout)
oc rollout restart deployment/<app>

# Check rollout status
oc rollout status deployment/<app>

# View ConfigMap
oc get configmap <name> -o yaml

# Update ConfigMap (replace)
oc create configmap <name> --from-literal=<KEY>=<value> --dry-run=client -o yaml | oc replace -f -

# Remote shell into pod
oc rsh <pod-name>

# Check pod environment variables
oc rsh <pod-name> env
```

---

## ConfigMap Injection Patterns

### Method 1: Inject All Keys from ConfigMap (Used in This Exercise)
```bash
oc set env deployment/myapp --from=configmap/myconfig
```

All keys in `myconfig` become environment variables.

### Method 2: Inject Specific Keys
```bash
oc set env deployment/myapp \
  MY_VAR=configmap:myconfig:key1 \
  ANOTHER_VAR=configmap:myconfig:key2
```

### Method 3: Mount ConfigMap as Volume
```yaml
spec:
  volumes:
    - name: config-volume
      configMap:
        name: myconfig
  containers:
    - name: myapp
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config
```

Files appear in `/etc/config/` with key names as filenames.

---

## Cleanup (After Exercise)
```bash
oc delete project acid
```

---