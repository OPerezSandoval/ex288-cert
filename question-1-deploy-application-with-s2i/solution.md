```markdown
## Solution (Timed Exercise - 15 minutes)

### Step 1: Create Project
```bash
oc new-project crimson
```

### Step 2: Deploy Application with S2I

```bash
oc new-app nodejs:latest~https://git.ocp4.example.com/developer/pastebin.git#master \
  --name=pastebin \
  --build-env npm_config_registry=http://nexus-infra.apps.ocp4.example.com/repository/npm/
```

### Step 3: Watch Build Fail (Expected)

```bash
oc logs -f bc/pastebin
```

Expected error:
```
npm ERR! JSON.parse Failed to parse json
npm ERR! Unexpected token in package.json
```

This is correct. The build should fail due to broken `package.json`.

---

### Step 4: Clone and Inspect

```bash
cd ~
git clone https://git.ocp4.example.com/developer/pastebin.git
cd pastebin
cat package.json
```

### Step 5: Validate and Find Error

```bash
python -m json.tool package.json
```

Output shows:
```
Expecting ',' delimiter: line 3 column 3 (char 44)
```

### Step 6: Fix the Error

```bash
vi package.json
```

Add comma after `"version": "1.0.0"`:
```json
{
  "name": "pastebin",
  "version": "1.0.0",   <-- Add comma here
  "description": "Simple pastebin app",
  ...
}
```

Save with `:wq`

### Step 7: Verify Fix

```bash
python -m json.tool package.json
# Should parse successfully now (no errors)

cat package.json  # Visual confirmation
```

### Step 8: Commit and Push

```bash
git add .
git commit -m "Fix missing comma in package.json"
git push origin master
```

### Step 9: Rebuild

```bash
oc start-build pastebin
oc logs -f bc/pastebin
```

Wait for:
```
Push successful
```

### Step 10: Check Deployment

```bash
oc get pods
# Wait until pastebin-1-xxxxx shows Running
```

### Step 11: Expose Service

```bash
oc expose svc/pastebin --hostname=pastebin-crimson.apps.ocp4.example.com
```

### Step 12: Verify Route

```bash
oc get route
curl http://pastebin-crimson.apps.ocp4.example.com
```

### Step 13: Test Application (Browser)

Open: `http://pastebin-crimson.apps.ocp4.example.com`

Add the required paste:
```
Per aspera ad astra
```

Click Submit.

### Step 14: Verify via API

```bash
curl http://pastebin-crimson.apps.ocp4.example.com/api/pastes | jq
```

Expected output:
```json
[
  {
    "id": 1,
    "text": "Per aspera ad astra"
  }
]
```

---

## Success Criteria

- Project `crimson` exists
- Application `pastebin` is running
- Route `pastebin-crimson.apps.ocp4.example.com` is accessible
- Pastebin entry "Per aspera ad astra" exists in `/api/pastes`

---

## Key Commands Reference

```bash
# Create project
oc new-project <name>

# Deploy with S2I (specific branch)
oc new-app <builder>~<git-url>#<branch> --name=<app> --build-env <VAR>=<value>

# Watch build logs
oc logs -f bc/<app>

# Restart build
oc start-build <app>

# Expose service
oc expose svc/<app> --hostname=<hostname>

# Validate JSON
python -m json.tool <file.json>

# Check pods
oc get pods

# Check routes
oc get route

# Check build status
oc get builds
```

---

## Cleanup (After Exercise)

```bash
oc delete project crimson
cd ~
rm -rf pastebin
```
```
