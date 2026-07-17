## Solution Option 1: Using Terminal Commands (Recommended for Exam)

### Step 1: Switch to the octane Project
```bash
oc project blogger-app
```

---

### Step 2: Set Liveness Probe Using CLI
```bash
oc set probe deployment/blogger \
  --liveness \
  --open-tcp=8080 \
  --initial-delay-seconds=10 \
  --timeout-seconds=30
```

**This command:**
- Sets a liveness probe on the `blogger` deployment
- Uses TCP socket check on port 8080
- Sets initial delay to 10 seconds
- Sets timeout to 30 seconds

---

### Step 3: Verify the Probe was Added
```bash
oc describe deployment blogger | grep -A 10 "Liveness"
```

**Expected output:**
```
Liveness:     tcp-socket :8080 delay=10s timeout=30s period=10s #success=1 #failure=3
```

**Or check the YAML:**
```bash
oc get deployment blogger -o yaml | grep -A 10 livenessProbe
```

**Expected output:**
```yaml
livenessProbe:
  tcpSocket:
    port: 8080
  initialDelaySeconds: 10
  timeoutSeconds: 30
  periodSeconds: 10
  successThreshold: 1
  failureThreshold: 3
```

---

### Step 4: Wait for Rollout

The probe change triggers a new rollout automatically:
```bash
oc rollout status deployment/blogger
```

**Verify pods are running:**
```bash
oc get pods
```

---

### Step 5: Verify Probe is Working
```bash
# Check pod events
oc describe pod -l deployment=blogger | grep -i liveness

# Check if pod is healthy
oc get pods -l deployment=blogger
```

**Expected status:** `Running` with `1/1` ready (not restarting)

---

### Step 6: Verify Changes Survive a Rebuild
```bash
# Trigger a new build
oc start-build blogger --follow

# Wait for new deployment
oc rollout status deployment/blogger

# Verify probe is still there
oc describe deployment blogger | grep -A 5 "Liveness"
```

**Probe should still be present after rebuild.**

---

## Solution Option 2: Using Web Console

### Step 1: Log in to OpenShift Web Console

1. Open browser: `https://console-openshift-console.apps.ocp4.example.com`
2. Login as `developer` / `developer`

---

### Step 2: Navigate to the blogger Deployment

1. Switch to **Developer** perspective (top-left dropdown)
2. Select project: **blogger-app**
3. Click **Topology** in left menu
4. Click on the **blogger** deployment (the circle/node)
5. In the right panel, click on the **blogger** deployment name (under Resources)

---

### Step 3: Edit Deployment to Add Liveness Probe

**Option A: Using the Form Editor**

1. Click **Actions** dropdown (top-right)
2. Select **Edit Health Checks**
3. Click **Add Liveness Probe**
4. Configure:
   - **Type:** TCP Socket
   - **Port:** 8080
   - **Initial Delay:** 10 seconds
   - **Timeout:** 30 seconds
5. Click **Save**

---

**Option B: Using YAML Editor**

1. Click **Actions** dropdown (top-right)
2. Select **Edit Deployment**
3. Switch to **YAML** tab
4. Find the `containers:` section
5. Add the liveness probe under the container spec:
```yaml
spec:
  containers:
    - name: blog
      image: ...
      ports:
        - containerPort: 8080
      livenessProbe:
        tcpSocket:
          port: 8080
        initialDelaySeconds: 10
        timeoutSeconds: 30
```

6. Click **Save**

---

### Step 4: Verify Rollout in Web Console

1. Go back to **Topology** view
2. Watch the **blogger** pod restart with the new configuration
3. Wait until the ring around the deployment turns **dark blue** (running)

---

## Success Criteria

- Project `blogger-app` exists
- Application `blogger` is running
- Deployment `blogger` has a liveness probe configured
- Probe uses TCP socket check on port 8080
- Probe has initialDelaySeconds: 10
- Probe has timeoutSeconds: 30
- Pods are not restarting due to failed probes
- Liveness probe persists after rebuilds

---

## Key Commands Reference
```bash
# Set liveness probe (TCP socket)
oc set probe deployment/<name> --liveness --open-tcp=<port> \
  --initial-delay-seconds=<seconds> --timeout-seconds=<seconds>

# Set liveness probe (HTTP GET)
oc set probe deployment/<name> --liveness --get-url=http://:8080/health \
  --initial-delay-seconds=<seconds> --timeout-seconds=<seconds>

# Set readiness probe
oc set probe deployment/<name> --readiness --open-tcp=<port> \
  --initial-delay-seconds=<seconds> --timeout-seconds=<seconds>

# Remove liveness probe
oc set probe deployment/<name> --liveness --remove

# View probe configuration
oc describe deployment <name> | grep -A 10 "Liveness"
oc get deployment <name> -o yaml | grep -A 10 livenessProbe

# Check pod status
oc get pods
oc describe pod <pod-name>

# Check probe failures in events
oc get events --sort-by='.lastTimestamp' | grep -i liveness
```

---

## Understanding Liveness vs Readiness Probes

### Liveness Probe
- **Purpose:** Detect if container is alive/healthy
- **Action on failure:** Kubernetes restarts the container
- **Use case:** App is stuck, deadlocked, or unresponsive
- **Example:** App crashes but container keeps running

### Readiness Probe
- **Purpose:** Detect if container is ready to serve traffic
- **Action on failure:** Remove pod from service endpoints (no traffic sent)
- **Use case:** App is starting up, warming up, or temporarily unavailable
- **Example:** Database connection not ready yet

### Startup Probe (Optional)
- **Purpose:** Give slow-starting apps more time
- **Action on failure:** Disable liveness/readiness until startup succeeds
- **Use case:** Legacy apps with long initialization

---

## Probe Types

### 1. TCP Socket Check (Used in This Exercise)
```bash
oc set probe deployment/blog --liveness --open-tcp=8080
```

Checks if port is open (can establish TCP connection).

### 2. HTTP GET
```bash
oc set probe deployment/blog --liveness --get-url=http://:8080/health
```

Sends HTTP GET request, expects 200-399 status code.

### 3. Exec Command
```bash
oc set probe deployment/blog --liveness --exec='cat /tmp/health'
```

Runs command in container, expects exit code 0.

---

## Common Probe Parameters
```bash
--initial-delay-seconds=10   # Wait before first probe
--timeout-seconds=30         # How long to wait for response
--period-seconds=10          # How often to probe (default: 10s)
--success-threshold=1        # Consecutive successes needed (default: 1)
--failure-threshold=3        # Consecutive failures before action (default: 3)
```

---

## Additional Practice

**Try these variations:**

### 1. Add Readiness Probe
```bash
oc set probe deployment/blogger --readiness --open-tcp=8080 \
  --initial-delay-seconds=5 --timeout-seconds=10
```

### 2. Use HTTP Health Check
```bash
# First ensure app has /health endpoint
oc set probe deployment/blogger --liveness --get-url=http://:8080/health \
  --initial-delay-seconds=10 --timeout-seconds=30
```

### 3. Test Probe Failure
```bash
# Change port to invalid value (will cause restarts)
oc set probe deployment/blogger --liveness --open-tcp=9999

# Watch pod restart
oc get pods -w

# Fix it
oc set probe deployment/blogger --liveness --open-tcp=8080
```

---

## Cleanup (After Exercise)
```bash
oc delete project blogger-app
cd ~
rm -rf blogger
```

---

