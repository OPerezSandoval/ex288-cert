# Question 4: Inject Configuration Data with ConfigMap

## Question

Using the `openshift/hello-openshift` image, deploy an application that meets the following requirements:

- The application is part of a project named: **acid**
- The application is named: **phosphoric**
- The application looks for a key named: **RESPONSE**
- The configuration is stored in a ConfigMap named: **sedicen**
- Once deployed, the application is running and available at `http://phosphoric-acid.apps.ocp4.example.com` and displays the following text:
```
  If you can see this your configmap works 
```
- If the ConfigMap is changed and the application is redeployed, then the changes in the ConfigMap are reflected in the new instance of the application

---

## Environment Setup

### Step 1: Understanding the hello-openshift Image

The `openshift/hello-openshift` image:
- Is a minimal test container
- Listens on port 8080 and 8888
- Returns the value of the `RESPONSE` environment variable as HTTP response
- Default response is "Hello OpenShift!" if no `RESPONSE` variable is set
- Perfect for testing ConfigMap injection

**No repository setup needed** - this question uses a pre-built container image.

---

### Step 2: Verify Access to OpenShift Cluster
```bash
oc login -u developer -p developer https://api.ocp4.example.com:6443
oc version
```

---