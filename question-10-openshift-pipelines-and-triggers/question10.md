# Question 10: Build and Deploy with OpenShift Pipelines (Tekton) and Triggers

## Question

Using the source code from `https://git.ocp4.example.com/developer/pipeline-app.git`, create a
CI/CD pipeline with OpenShift Pipelines (Tekton) that meets the following requirements:

- The pipeline is part of a project named: **cicd**
- The pipeline is named: **build-and-deploy**
- The pipeline uses a **workspace named `shared-data` that is backed by a PVC** so that all
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

**Note — GitLab webhooks to the cluster are already permitted.** The classroom GitLab has been
configured to allow outbound webhook requests to the local network, so the EventListener route is
accepted as a webhook URL. See Step 6 of the setup below.

**Note — Git credentials are already configured.** The `pipeline-app` repository is **private** and
the lab GitLab is served over **HTTPS with a self-signed certificate**. A secret named
**`git-basic-auth`** already exists in the `cicd` project holding the `.gitconfig` and
`.git-credentials` files for `https://git.ocp4.example.com`. You must **bind it to the `git-clone`
task's `basic-auth` workspace** — the clone fails without it. See Step 5 of the setup below for what
the secret contains.

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

The `pipeline-app` repo is a minimal static web app plus a `Dockerfile`.
For a self-hosted lab, create it in GitLab (`https://git.ocp4.example.com`) with these files:

**`Dockerfile`** — the file **must be named `Dockerfile`**; the shipped `buildah` task defaults its
`DOCKERFILE` param to `./Dockerfile`, and a repo containing only a `Containerfile` fails the build
unless you override that param.
```dockerfile
FROM registry.access.redhat.com/ubi8/ubi-minimal:latest
COPY index.html /usr/share/app/index.html
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
git clone https://git.ocp4.example.com/developer/pipeline-app.git
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

### Step 5: Create the Git credentials secret (lab setup — pre-created on the exam)

The repository is private and the GitLab certificate is self-signed. The `git-clone` task consumes
credentials through its optional **`basic-auth` workspace**, which expects a secret containing a
`.gitconfig` and a `.git-credentials` file.

```bash
cat <<'EOF' | oc apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: git-basic-auth
  namespace: cicd
type: Opaque
stringData:
  .gitconfig: |
    [credential "https://git.ocp4.example.com"]
        helper = store
    [http]
        sslVerify = false
  .git-credentials: |
    https://developer:d3v3lop3r@git.ocp4.example.com
EOF
```

> Replace `developer:d3v3lop3r` with the lab's actual GitLab credentials. If a password contains
> `@`, `:`, `/`, or `#`, percent-encode it inside `.git-credentials`.

---

### Step 6: Allow GitLab webhooks to reach the cluster (lab setup — pre-configured on the exam)

The classroom GitLab runs on its own VM (`192.168.50.50`), and `apps.ocp4.example.com` resolves to
an RFC1918 address. GitLab blocks webhooks to the local network by default and reports it in the UI
as **"invalid URL"** when you try to save the hook. An admin must allow it once, before students
start.

```bash
# from workstation
ssh 192.168.50.50
su -                                          # classroom root password: redhat

# GitLab's admin account is 'root'; its password is here while unchanged:
cat /etc/gitlab/initial_root_password
# if that no longer works, reset it:
gitlab-rake "gitlab:password:reset[root]"
```

Log in to `https://git.ocp4.example.com` as **`root`**, then go to
**Admin Area → Settings → Network → Outbound requests**
(`https://git.ocp4.example.com/admin/application_settings/network`) and enable:

- *Allow requests to the local network from webhooks and integrations*
- *Allow requests to the local network from system hooks*

**Save changes.** Students can now add a webhook pointing at the EventListener route without
hitting the "invalid URL" rejection.

---

## What you must deliver

1. A PVC-backed workspace (`shared-data`) shared by every task.
2. A `Pipeline/build-and-deploy` with `fetch-repository` → `build-image` → `deploy`.
3. The `git-basic-auth` secret bound to the `git-clone` task's `basic-auth` workspace, and
   `SSL_VERIFY: "false"` set on `fetch-repository` for the self-signed certificate.
4. A successful manual `PipelineRun`.
5. Triggers (`TriggerTemplate` + `TriggerBinding` + `EventListener`) plus RBAC (a RoleBinding **and**
   a ClusterRoleBinding), an exposed route, and a GitLab webhook so a `git push` starts a run
   automatically.
6. The app reachable at `http://pipeline-app-cicd.apps.ocp4.example.com`.
