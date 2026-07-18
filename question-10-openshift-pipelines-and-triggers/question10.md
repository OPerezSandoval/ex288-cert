# Question 10: Build and Deploy with OpenShift Pipelines (Tekton) and Triggers

## Question

Using the source code from `http://git.ocp4.example.com/developer/pipeline-app.git`, create a
CI/CD pipeline with OpenShift Pipelines (Tekton) that meets the following requirements:

- The pipeline is part of a project named: **cicd**
- The pipeline is named: **build-and-deploy**
- The pipeline uses a single **workspace named `shared-data` that is backed by a PVC** so that all
  tasks share the same cloned source code
- The pipeline runs the following tasks, **in order**:
  1. **fetch-repository** — clones the Git repository into the shared workspace (uses `git-clone`)
  2. **build-image** — builds a container image from the cloned source with `buildah` and pushes it
     to the internal registry as `image-registry.openshift-image-registry.svc:5000/cicd/pipeline-app:latest`
  3. **deploy** — deploys/rolls out the application using `openshift-client`
- A **PipelineRun** can be started manually (with `tkn` or a YAML manifest) and completes successfully
- A **push to the Git repository automatically starts a new PipelineRun** using a
  **TriggerTemplate, TriggerBinding, and EventListener** (webhook-driven)
- Once the pipeline succeeds, the application is running and available at
  `http://pipeline-app-cicd.apps.ocp4.example.com` and returns `Hello, Pipelines!`

**Note:** The Red Hat OpenShift Pipelines Operator is already installed on the cluster. The pipeline
service account must be able to push to the internal registry and create application resources.

---

## Environment Setup

### Step 1: Verify the OpenShift Pipelines Operator is installed
```bash
oc login -u developer -p developer https://api.ocp4.example.com:6443

# Tekton CRDs should exist
oc get crd | grep tekton
# pipelines.tekton.dev, tasks.tekton.dev, pipelineruns.tekton.dev,
# triggertemplates.triggers.tekton.dev, eventlisteners.triggers.tekton.dev ...

# The tkn CLI should be available
tkn version
```

If the operator is not installed (lab setup only), an admin installs it:
```bash
cat <<'EOF' | oc apply -f -
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: openshift-pipelines-operator-rh
  namespace: openshift-operators
spec:
  channel: latest
  name: openshift-pipelines-operator-rh
  source: redhat-operators
  sourceNamespace: openshift-marketplace
EOF
```

---

### Step 2: Create the cicd project
```bash
oc new-project cicd
```

---

### Step 3: Seed the Git repository (lab setup)

The `pipeline-app` repo is a minimal Go/Node/quarkus-style web app plus a `Containerfile`.
For a self-hosted lab, create it in GitLab (`http://git.ocp4.example.com`) with these files:

**`Containerfile`**
```dockerfile
FROM registry.access.redhat.com/ubi8/ubi-minimal:latest
COPY index.html /usr/share/app/index.html
# Tiny static server using busybox httpd-style; replace with your runtime as needed
RUN microdnf install -y python3 && microdnf clean all
WORKDIR /usr/share/app
EXPOSE 8080
USER 1001
CMD ["python3", "-m", "http.server", "8080"]
```

**`index.html`**
```html
Hello, Pipelines!
```

Then push:
```bash
git clone http://git.ocp4.example.com/developer/pipeline-app.git
cd pipeline-app
# add the two files above
git add . && git commit -m "seed pipeline-app" && git push
```

---

### Step 4: Confirm the cluster tasks / resolvers are available
```bash
# OpenShift Pipelines ships resolvable cluster tasks in the openshift-pipelines namespace
oc get task -n openshift-pipelines | grep -E 'git-clone|buildah|openshift-client'

# On newer versions these are consumed via the cluster resolver instead of ClusterTask:
#   resolver: cluster  ->  name: git-clone, namespace: openshift-pipelines
```

---

## What you must deliver

1. A PVC-backed workspace (`shared-data`) shared by every task.
2. A `Pipeline/build-and-deploy` with `fetch-repository` → `build-image` → `deploy`.
3. A successful manual `PipelineRun`.
4. Triggers (`TriggerTemplate` + `TriggerBinding` + `EventListener`) plus an exposed route and a
   GitLab webhook so a `git push` starts a run automatically.
5. The app reachable at `http://pipeline-app-cicd.apps.ocp4.example.com`.
