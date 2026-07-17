## Solution (Timed Exercise - 15 minutes)

### Step 1: Login as Cluster Admin
```bash
oc login -u admin -p admin https://api.ocp4.example.com:6443
```

---

### Step 2: Enable Default Route on Image Registry
```bash
oc patch configs.imageregistry.operator.openshift.io/cluster \
  --patch '{"spec":{"defaultRoute":true}}' \
  --type=merge
```
or you can do it trough the ui 

**This command:**
- Enables the default route for the internal image registry
- Makes the registry accessible from outside the cluster

---

### Step 3: Verify Route Was Created
```bash
oc get route -n openshift-image-registry
```

**Expected output:**
```
NAME            HOST/PORT                                              PATH   SERVICES         PORT    TERMINATION   WILDCARD
default-route   default-route-openshift-image-registry.apps.ocp4...          image-registry   <all>   reencrypt     None
```

**Get the registry route:**
```bash
REGISTRY_ROUTE=$(oc get route default-route -n openshift-image-registry -o jsonpath='{.spec.host}')
echo $REGISTRY_ROUTE
```

**Expected output:**
```
default-route-openshift-image-registry.apps.ocp4.example.com
```

---

### Step 4: Get Authentication Token
```bash
TOKEN=$(oc whoami -t)
echo $TOKEN
```

**Expected output:** Long token string (e.g., `sha256~abc123...`)

---

### Step 6: Login to Registry with Podman
```bash
podman login -u admin -p ${TOKEN} ${REGISTRY_ROUTE} --tls-verify=false
```

**Expected output:**
```
Login Succeeded!
```

---

### Step 7: Verify Image Exists in Registry
```bash
# List images in project
oc get is
```

**Expected output:**
```
NAME         IMAGE REPOSITORY                                                      TAGS      UPDATED
sample-image   default-route-openshift-image-registry.apps.ocp4.example.com/mi...   latest    10 minutes ago
```

---

### Step 8: Pull the Image with Podman
```bash
podman pull ${REGISTRY_ROUTE}/registry-practice/sample-image:latest --tls-verify=false
```

---

### Step 9: Verify Image Was Pulled
```bash
podman images | grep sample-image
```

**Expected output:**
```
default-route-openshift-image-registry.apps.ocp4.example.com/minzeoul/chartreuse   latest   abc123def456   10 minutes ago   107 MB
```
---

## Success Criteria

- Internal image registry has `defaultRoute: true` enabled
- Route `default-route` exists in `openshift-image-registry` namespace
- Podman successfully logged in to the registry
- Image `sample-image` from project `registry-practice` was successfully pulled
- `podman images` shows the pulled image

---

## Key Commands Reference
```bash
# Enable registry default route
oc patch configs.imageregistry.operator.openshift.io/cluster \
  --patch '{"spec":{"defaultRoute":true}}' \
  --type=merge

# Check registry configuration
oc get configs.imageregistry.operator.openshift.io/cluster -o yaml

# Get registry route
oc get route -n openshift-image-registry

# Get authentication token
oc whoami -t

# Login to registry with podman
podman login -u <user> -p <token> <registry-route> --tls-verify=false

# Pull image from registry
podman pull <registry-route>/<project>/<image>:<tag> --tls-verify=false

# List local images
podman images

# Inspect image
podman inspect <image>

# Remove local image
podman rmi <image>
```

---

## Understanding OpenShift Image Registry

### Internal vs External Access

**Internal Access (within cluster):**
```
image-registry.openshift-image-registry.svc:5000
```
- Used by pods and builds within the cluster
- No external route needed

**External Access (outside cluster):**
```
default-route-openshift-image-registry.apps.ocp4.example.com
```
- Requires `defaultRoute: true`
- Accessible from outside the cluster via HTTPS
- Used by external tools (podman, docker, skopeo)

---

## Common Issues and Troubleshooting

| Issue | Symptom | Fix |
|-------|---------|-----|
| **Route not created** | `oc get route -n openshift-image-registry` shows nothing | Wait 1-2 minutes after patching; operator needs time to create route |
| **Login fails** | `podman login` returns authentication error | Verify token: `oc whoami -t`; ensure user has registry access |
| **TLS certificate error** | Certificate verification failed | Use `--tls-verify=false` flag with podman |
| **Image not found** | `Error: image not known` | Verify imagestream exists: `oc get is -n minzeoul` |
| **Permission denied** | User cannot pull image | Grant registry-viewer role: `oc policy add-role-to-user registry-viewer <user>` |
| **defaultRoute still false** | Patch didn't apply | Check operator status: `oc get co image-registry` |

**If route doesn't appear after patch:**
```bash
# Check image registry operator status
oc get clusteroperator image-registry

# Check operator logs
oc logs -n openshift-image-registry deployment/cluster-image-registry-operator

# Verify patch was applied
oc get configs.imageregistry.operator.openshift.io/cluster -o yaml | grep defaultRoute

# Force operator sync
oc delete pod -n openshift-image-registry -l name=cluster-image-registry-operator
```
---
