# Question 9: Open the Internal Image Registry

## Question

Expose the OpenShift internal image registry
Create a sample image from the registry 
Pull it with podman 

**Note:** Your OpenShift user has been granted all the necessary roles to complete these tasks.

---

## Environment Setup

### Step 1: Create the registry-practice Project
```bash
oc login -u developer -p developer https://api.ocp4.example.com:6443

oc new-project registry-practice
```

---

### Step 2: Deploy a Sample Application (chartreuse)

We'll use an existing cluster image (ubi8-minimal) and tag it as `chartreuse`:
```bash
# Use an existing UBI minimal image from Red Hat
oc import-image sample-image:latest \
  --from=registry.access.redhat.com/ubi8/ubi-minimal:latest \
  --confirm \

# Verify imagestream was created
oc get is image-sample
```

**Alternative - Deploy a simple app:**
```bash
oc new-app --name=image-sample \
  --docker-image=registry.access.redhat.com/ubi8/ubi-minimal:latest \
  
# Wait for deployment
oc get pods 
```

---

### Step 4: Verify Image Registry Operator
```bash
# Check image registry operator status
oc get configs.imageregistry.operator.openshift.io/cluster -o yaml
```
---

