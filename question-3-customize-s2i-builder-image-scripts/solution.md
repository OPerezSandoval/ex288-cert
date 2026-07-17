## Solution (Timed Exercise - 20 minutes)

### Step 1: Verify Namespaces

```bash
# Ensure build namespace exists
oc project s2i-builds

# Create the deployment namespace
oc new-project tocin
```

---

### Step 2: Update Local Repository

```bash
cd ~/oxy
git pull origin master
```

---

### Step 3: Customize the Assemble Script

```bash
vi .s2i/bin/assemble
```

**Replace with this customized version:**

```bash
#!/bin/bash

echo "Running custom assemble script..."

# Copy all HTML files from /tmp/src to current directory
cp -Rf /tmp/src/*.html ./

# Generate build date in YYYY-mm-dd format
DATE=$(date "+%F")

# Create info.html with date and required text
echo "$DATE" > ./info.html
echo "Astra inclinant, sed non obligant" >> ./info.html

# Call the original assemble script if it exists
if [ -f /usr/libexec/s2i/assemble ]; then
  /usr/libexec/s2i/assemble
fi
```

**Key additions:**
- `cp -Rf /tmp/src/*.html ./` - copies HTML files
- `DATE=$(date "+%F")` - generates date in YYYY-mm-dd format
- Creates `info.html` with date and required text

Save with `:wq`

---

### Step 4: Verify Script is Executable

```bash
chmod +x .s2i/bin/assemble
ls -la .s2i/bin/assemble
```

Should show `-rwxr-xr-x` permissions.

---

### Step 5: Commit and Push Changes

```bash
git add .
git commit -m "Customize assemble script to copy HTML and generate info.html"
git push origin master
```

---

### Step 6: Build S2I Image in s2i-builds Namespace

```bash
# Switch to build namespace
oc project s2i-builds

# Create the S2I build
oc new-app httpd:2.4~https://git.ocp4.example.com/developer/oxy.git#master \
  --name=oxy \
  --strategy=source

# Watch the build
oc logs -f bc/oxy
```

**Look for:**
```
Running custom assemble script...
```

This confirms your custom assemble script is executing.

**Wait for build to complete:**

```bash
oc get builds
```

---

### Step 7: Grant Access for tocin Namespace to Pull Image

The `tocin` namespace needs permission to pull the image from `s2i-builds`:

```bash
# Allow default service account in tocin to pull from s2i-builds
oc policy add-role-to-user system:image-puller \
  system:serviceaccount:tocin:default \
  --namespace=s2i-builds
```

---

### Step 8: Deploy Application in tocin Namespace

```bash
# Switch to deployment namespace
oc project tocin

# Deploy using the image from s2i-builds namespace
oc new-app s2i-builds/oxy:latest --name=oxy
```

**Wait for deployment:**

```bash
oc get pods
```

Wait until `oxy-xxxxx` shows `Running`.

---

### Step 9: Expose Service

```bash
oc expose svc/oxy --hostname=oxy-tocin.apps.ocp4.example.com
```

---

### Step 10: Verify Main Page

```bash
curl http://oxy-tocin.apps.ocp4.example.com
```

**Expected output:**

```html
<!DOCTYPE html>
<html>
<head>
  <title>Oxy Application</title>
</head>
<body>
  <h1>Provided text</h1>
</body>
</html>
```

---

### Step 11: Verify Info Page

```bash
curl http://oxy-tocin.apps.ocp4.example.com/info.html
```

**Expected output:**

```
2025-10-11
Astra inclinant, sed non obligant
```

(Date will be the current date in YYYY-mm-dd format)

---

### Step 12: Verify Cross-Namespace Image Reference

```bash
# Check that the deployment uses the s2i-builds image
oc get deployment oxy -o jsonpath='{.spec.template.spec.containers[0].image}'
```

**Expected output:**
```
image-registry.openshift-image-registry.svc:5000/s2i-builds/oxy@sha256:...
```

This confirms the image is being pulled from the `s2i-builds` namespace.

---

### Step 13: Verify in Browser

Open both URLs in browser:
- `http://oxy-tocin.apps.ocp4.example.com` 
- `http://oxy-tocin.apps.ocp4.example.com/info.html` 

---

## Success Criteria

- Namespace `s2i-builds` exists and contains the build
- Namespace `tocin` exists and contains the deployment
- Application `oxy` is running in `tocin` namespace
- Image was built in `s2i-builds` namespace
- Deployment in `tocin` successfully pulls image from `s2i-builds`
- Route `oxy-tocin.apps.ocp4.example.com` is accessible
- Main page displays "Amor vincit omnia"
- `/info.html` displays build date in YYYY-mm-dd format
- `/info.html` displays "Astra inclinant, sed non obligant"
- Custom `.s2i/bin/assemble` script successfully copied HTML files and generated info.html

---

## Key Commands Reference

```bash
# Create project
oc new-project <name>

# Deploy with S2I in specific namespace
oc new-app <builder>~<git-url>#<branch> --name=<app> --strategy=source

# Grant cross-namespace image pull access
oc policy add-role-to-user system:image-puller \
  system:serviceaccount:<target-ns>:default \
  --namespace=<source-ns>

# Deploy from another namespace's imagestream
oc new-app <source-namespace>/<imagestream>:<tag> --name=<app>

# Watch build logs
oc logs -f bc/<app>

# Check image reference
oc get deployment <app> -o jsonpath='{.spec.template.spec.containers[0].image}'

# Expose service
oc expose svc/<app> --hostname=<hostname>
```

---

## Common Issues and Troubleshooting

| Issue | Symptom | Fix |
|-------|---------|-----|
| **Image pull error** | `ErrImagePull` in tocin namespace | Grant `system:image-puller` role to tocin's default SA |
| **Assemble script not executing** | Custom code doesn't run | Ensure script is executable: `chmod +x .s2i/bin/assemble` |
| **Files not copied** | HTML files missing in pod | Verify `cp -Rf /tmp/src/*.html ./` in assemble script |
| **Date format wrong** | Date not in YYYY-mm-dd | Use `date "+%F"` or `date "+%Y-%m-%d"` |
| **info.html not generated** | 404 on /info.html | Check assemble logs: `oc logs bc/oxy -n s2i-builds` |
| **Wrong namespace** | Build or deploy fails | Verify current namespace: `oc project` |

**If cross-namespace pull fails:**

```bash
# Check role binding
oc get rolebinding -n s2i-builds | grep image-puller

# Re-apply permission
oc policy add-role-to-user system:image-puller \
  system:serviceaccount:tocin:default \
  --namespace=s2i-builds

# Check deployment events
oc describe deployment oxy -n tocin

# Force new rollout
oc rollout restart deployment/oxy -n tocin
```

---

## Cleanup 

```bash
oc delete project s2i-builds
oc delete project tocin
cd ~
rm -rf oxy
```
