## Solution (Timed Exercise - 15 minutes)

### Step 1: Create Project

```bash
oc new-project container-build
```

### Step 2: Clone and Inspect

```bash
cd ~
git clone https://git.ocp4.example.com/developer/build.git
cd build
cat Dockerfile
```

### Step 3: Count Current Layers (Before Optimization)

```bash
grep -E '^(FROM|RUN|COPY|ADD|LABEL|MAINTAINER)' Dockerfile | wc -l
```

**Current layer count:** Around 8-9 layers (needs optimization)

---

### Step 4: Optimize the Dockerfile

```bash
vi Dockerfile
```

**Apply these optimizations:**

1. **Combine multiple RUN statements into one** (reduces layers)
2. **Combine multiple LABEL statements** (reduces layers)
3. **Add ONBUILD instruction** to allow child image overrides

**Optimized Dockerfile:**

```dockerfile
FROM registry.access.redhat.com/ubi8/ubi-minimal:latest

LABEL version="1.0" \
      description="this is Dockerfile" \
      maintainer="Red Hat Training <training@redhat.com>"

USER root

RUN microdnf install -y python3 && \
    microdnf clean all && \
    mkdir -p /app && \
    echo "Hello container!" > /app/index.html

ENV DOCROOT=/app

ONBUILD COPY src/ ${DOCROOT} <- or you can use /app directly 

EXPOSE 8080

USER 1001

WORKDIR /app

CMD ["python3", "-m", "http.server", "8080"]
```

---

### Step 5: Verify Layer Count (After Optimization)

```bash
grep -E '^(FROM|RUN|COPY|ADD|LABEL|ONBUILD)' Dockerfile | wc -l
```

**New layer count:** Should be 4-5 layers (well under 7)

---

### Step 6: Commit and Push Optimized Dockerfile

```bash
git add Dockerfile
git commit -m "Optimize Dockerfile - combine RUN/LABEL, add ONBUILD"
git push origin master
```

---

### Step 7: Build Image Using Binary Build

```bash
oc new-app --name container-build https://git.link 
or
oc new-build --name=container-build --strategy=docker --binary=true
oc start-build container-build --from-dir=. --follow
```

**Wait for build to complete:**

```bash
oc get builds
```

---

### Step 8: Deploy the Built Image

```bash
oc new-app container-build --name=container-build
```

**Wait for deployment:**

```bash
oc get pods
```

---

### Step 9: Expose Service

```bash
oc expose svc/container-build --hostname=app-optimize.apps.ocp4.example.com
```

---

### Step 10: Verify Application

```bash
oc get route
curl http://build-crimson.apps.ocp4.example.com
```

**Expected output:**

```html
Hello container!
```

---

### Step 11: Test ONBUILD Functionality (Child Image)

**Create a child image to verify ONBUILD works:**

```bash
cd ~
mkdir child-test
cd child-test

# Create child Dockerfile
cat > Dockerfile <<EOF
FROM image-registry.openshift-image-registry.svc:5000/crimson/container-build:latest
EOF

# Create src/ directory with override content
mkdir src
cat > src/index.html <<'EOF'
<!DOCTYPE html>
<html>
<head><title>Child Override</title></head>
<body>
  <h1>Child Image Content</h1>
  <p>ONBUILD successfully copied src/ content!</p>
</body>
</html>
EOF

# Create new build for child image
oc new-build --name=container-build-child \
  --strategy=docker \
  --binary=true

# Start build
oc start-build container-build-child --from-dir=. --follow

# Deploy child
oc new-app container-build-child --name=container-build-child
oc expose svc/container-build-child --hostname=build-child-crimson.apps.ocp4.example.com

# Test
curl http://build-child-crimson.apps.ocp4.example.com
```

**Expected output:**

```html
<!DOCTYPE html>
<html>
<head><title>Child Override</title></head>
<body>
  <h1>Child Image Content</h1>
  <p>ONBUILD successfully copied src/ content!</p>
</body>
</html>
```

---

### Step 12: Verify Image Size and Layers

```bash
# Check imagestream
oc describe is/container-build

# Verify size is under 256 MiB
oc get is container-build -o jsonpath='{.status.tags[0].items[0].dockerImageMetadata.Size}' | awk '{print $1/1024/1024 " MiB"}'
```

---

## Success Criteria

- Project `container-build` exists
- Application `app-optimize` is running
- Route `app-optimize.apps.ocp4.example.com` is accessible and returns "Hello container!"
- Dockerfile has 7 or fewer layers
- Image size is under 256 MiB
- `ONBUILD COPY src/ ${DOCROOT}` instruction exists in Dockerfile
- Child images can override content by providing `src/` directory

---

## Key Commands Reference

```bash
# Count layers in Dockerfile
grep -E '^(FROM|RUN|COPY|ADD|LABEL)' Dockerfile | wc -l

# Create binary build
oc new-build --name=<name> --strategy=docker --binary=true

# Start build from directory
oc start-build <build-name> --from-dir=. --follow

# Create app from imagestream
oc new-app <imagestream>:<tag> --name=<app>

# Expose service
oc expose svc/<app> --hostname=<hostname>

# Check image size
oc get is <imagestream> -o jsonpath='{.status.tags[0].items[0].dockerImageMetadata.Size}'

# Describe imagestream
oc describe is/<imagestream>
```

---

## Optimization Techniques

### Layer Reduction:

1. **Combine RUN statements:**
   ```dockerfile
   # Bad (4 layers)
   RUN command1
   RUN command2
   RUN command3
   RUN command4
   
   # Good (1 layer)
   RUN command1 && \
       command2 && \
       command3 && \
       command4
   ```

2. **Combine LABEL statements:**
   ```dockerfile
   # Bad (3 layers)
   LABEL key1="value1"
   LABEL key2="value2"
   LABEL key3="value3"
   
   # Good (1 layer)
   LABEL key1="value1" \
         key2="value2" \
         key3="value3"
   ```

3. **Clean package cache in same RUN:**
   ```dockerfile
   RUN microdnf install -y package && \
       microdnf clean all
   ```

### Size Reduction:

- Use minimal base images (ubi-minimal instead of ubi)
- Clean package manager cache in same RUN statement
- Remove unnecessary files and build dependencies
- Use `.dockerignore` to exclude unnecessary files from build context


### Helpful commands 

```bash
# Check build logs
oc logs -f bc/container-build

# Verify Dockerfile syntax
podman build -t test .

# Check build config
oc describe bc/container-build
```

---

## Cleanup (After Exercise)

```bash
oc delete project crimson
cd ~
rm -rf build child-test
```
