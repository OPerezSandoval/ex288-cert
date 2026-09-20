## Solution (Timed Exercise — 45 minutes)

This is the longest task on the exam. Work top-to-bottom and **verify each object renders before
applying it**. The flow is: permissions → workspace PVC → git credentials → pipeline → manual
PipelineRun → triggers → webhook → verify.

---

### Step 1: Login and select the project
```bash
oc login -u developer -p developer https://api.ocp4.example.com:6443
oc project cicd
```

---

### Step 2: Grant the pipeline service account the permissions it needs

The `pipeline` service account is created automatically by the Pipelines operator in every project.
It needs to push to the internal registry and manage app resources.

```bash
# Allow the pipeline SA to push images to this project's registry area
oc policy add-role-to-user system:image-builder -z pipeline

# Allow it to create/rollout Deployments, Services, Routes in this project
oc policy add-role-to-user edit -z pipeline
```

> **Gotcha:** `buildah` builds run as the `pipeline` SA. Without `system:image-builder` the push in
> `build-image` fails with an authentication/permission error. Without `edit`, the `deploy` task's
> `oc new-app` / `oc apply` is forbidden.

---

### Step 3: Create the PVC that backs the shared workspace

Two valid approaches — the exam may ask for either. Know both.

**Option A — a static PersistentVolumeClaim** (referenced by name in the PipelineRun):
```bash
cat <<'EOF' | oc apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-data-pvc
  namespace: cicd
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF
```

**Option B — a `volumeClaimTemplate`** inside the PipelineRun (a fresh PVC per run, auto-cleaned).
Shown in Step 6. Prefer this when the exam says "each run should get its own storage."

> **Gotcha (RWO + parallel tasks):** a `ReadWriteOnce` PVC can only be mounted by pods on the **same
> node**. Since our tasks run sequentially (`runAfter`), RWO is fine. If you fan out parallel tasks
> that each mount the workspace, they may land on different nodes and fail to mount — use `runAfter`
> to serialize, or request RWX storage.

---

### Step 4: Confirm the Git credentials secret exists

The lab repository is **private** and served over **HTTPS with a self-signed certificate**. The
`git-basic-auth` secret is pre-created by the setup (see `question10.md` Step 5). Confirm it:

```bash
oc get secret git-basic-auth -n cicd
oc get secret git-basic-auth -n cicd -o jsonpath='{.data}' | python3 -m json.tool
# expect keys: .gitconfig  and  .git-credentials
```

If it is missing, create it:
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

> **Why a workspace and not a service-account secret?** Tekton can also inject credentials from a
> `kubernetes.io/basic-auth` secret annotated `tekton.dev/git-0: https://git.ocp4.example.com` and
> linked with `oc secrets link pipeline <secret>`. That path (creds-init) writes the credentials to
> `/tekton/home`, but the shipped `git-clone` task runs git with `HOME=/home/git` (its `USER_HOME`
> param default, backed by an `emptyDir` mounted at `/home/git`). The credentials land where git
> never looks, and the clone fails with `could not read Username`. The `basic-auth` workspace copies
> the files to `$USER_HOME` directly, so it works regardless of creds-init behavior. **Use the
> workspace.**

---

### Step 5: Create the Pipeline

Save as `pipeline.yaml`. This uses the **cluster resolver** to pull the shipped `git-clone`,
`buildah`, and `openshift-client` tasks from `openshift-pipelines`. (If your version still exposes
`ClusterTask`, replace each `taskRef` with `kind: ClusterTask` + `name:` — see the note at the end.)

```yaml
apiVersion: tekton.dev/v1
kind: Pipeline
metadata:
  name: build-and-deploy
  namespace: cicd
spec:
  params:
    - name: git-url
      type: string
      default: https://git.ocp4.example.com/developer/pipeline-app.git
    - name: git-revision
      type: string
      default: main
    - name: image
      type: string
      default: image-registry.openshift-image-registry.svc:5000/cicd/pipeline-app:latest
  workspaces:
    - name: shared-data          # <-- the shared source workspace (PVC-backed)
    - name: git-auth             # <-- the git credentials secret
  tasks:
    # 1) clone into the shared workspace
    - name: fetch-repository
      taskRef:
        resolver: cluster
        params:
          - name: kind
            value: task
          - name: name
            value: git-clone
          - name: namespace
            value: openshift-pipelines
      workspaces:
        - name: output
          workspace: shared-data
        - name: basic-auth       # <-- git-clone's credentials workspace
          workspace: git-auth
      params:
        - name: URL
          value: $(params.git-url)
        - name: REVISION
          value: $(params.git-revision)
        - name: DELETE_EXISTING
          value: "true"
        - name: SSL_VERIFY       # <-- self-signed lab certificate
          value: "false"

    # 2) build + push the image from the cloned source
    - name: build-image
      runAfter:
        - fetch-repository
      taskRef:
        resolver: cluster
        params:
          - name: kind
            value: task
          - name: name
            value: buildah
          - name: namespace
            value: openshift-pipelines
      workspaces:
        - name: source
          workspace: shared-data
      params:
        - name: IMAGE
          value: $(params.image)
        - name: DOCKERFILE
          value: ./Dockerfile    # <-- must match the filename in the repo

    # 3) deploy / roll out the app
    - name: deploy
      runAfter:
        - build-image
      taskRef:
        resolver: cluster
        params:
          - name: kind
            value: task
          - name: name
            value: openshift-client
          - name: namespace
            value: openshift-pipelines
      params:
        - name: SCRIPT
          value: |
            oc new-app --image=$(params.image) --name=pipeline-app || \
              oc set image deployment/pipeline-app pipeline-app=$(params.image)
            oc rollout status deployment/pipeline-app
            oc expose deployment/pipeline-app --port=8080 2>/dev/null || true
            oc expose service/pipeline-app --hostname=pipeline-app-cicd.apps.ocp4.example.com 2>/dev/null || true
```

Apply it:
```bash
oc apply -f pipeline.yaml
tkn pipeline list
```

> **Note on internal-registry image references:** because the image lives in the cluster registry,
> the `deploy` uses the in-cluster service DNS
> `image-registry.openshift-image-registry.svc:5000/cicd/pipeline-app:latest`. No external route or
> `--tls-verify` needed for the pull inside the cluster. The `buildah` push works with the task's
> default `TLS_VERIFY: "true"` because the internal registry presents the cluster's service CA.
> **The param is spelled `TLS_VERIFY`, not `TLSVERIFY`** — a misspelled param name is silently
> ignored, so you get the default instead of the value you thought you set.

---

### Step 6: (Optional) Inspect the shipped task params

Param names differ slightly by version. When unsure, read them off the cluster:
```bash
tkn task describe git-clone -n openshift-pipelines
tkn task describe buildah -n openshift-pipelines
oc get task git-clone -n openshift-pipelines \
  -o jsonpath='{range .spec.params[*]}{.name}{"\t"}{.default}{"\n"}{end}'
oc get task git-clone -n openshift-pipelines \
  -o jsonpath='{range .spec.workspaces[*]}{.name}{"\t"}{.optional}{"\n"}{end}'
```

---

### Step 7: Start a PipelineRun manually and confirm it succeeds

**Option A — declarative PipelineRun YAML (`pipelinerun.yaml`) — this is the tested one:**
```yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  generateName: build-and-deploy-run-
  namespace: cicd
spec:
  pipelineRef:
    name: build-and-deploy
  taskRunTemplate:
    serviceAccountName: pipeline
  params:
    - name: git-url
      value: https://git.ocp4.example.com/developer/pipeline-app.git
    - name: git-revision
      value: main
    - name: image
      value: image-registry.openshift-image-registry.svc:5000/cicd/pipeline-app:latest
  workspaces:
    - name: shared-data
      volumeClaimTemplate:            # fresh PVC per run
        spec:
          accessModes: [ReadWriteOnce]
          resources:
            requests:
              storage: 1Gi
      # --- or bind the static PVC from Step 3 instead ---
      # persistentVolumeClaim:
      #   claimName: shared-data-pvc
    - name: git-auth
      secret:
        secretName: git-basic-auth
```
Because it uses `generateName`, apply it with `create`, not `apply`:
```bash
oc create -f pipelinerun.yaml
```

> `taskRunTemplate.serviceAccountName` is the `tekton.dev/v1` spelling. On `v1beta1` it is a bare
> `spec.serviceAccountName`.

**Option B — with `tkn`:**
```bash
tkn pipeline start build-and-deploy \
  --param git-url=https://git.ocp4.example.com/developer/pipeline-app.git \
  --param git-revision=main \
  --workspace name=shared-data,claimName=shared-data-pvc \
  --workspace name=git-auth,secret=git-basic-auth \
  --showlog
```

**Watch it:**
```bash
tkn pipelinerun logs -f --last
tkn pipelinerun list
oc get pipelinerun
```
All three tasks must reach `Succeeded`. A healthy `fetch-repository` log shows:
```
---> Phase: Configuring Git authentication with 'basic-auth' Workspace files...
---> Phase: Copying '/workspace/basic-auth/.git-credentials' to '/home/git'...
```
If that copy destination is anything other than the step's `HOME`, the clone will still prompt for
credentials — see the `USER_HOME` note in Step 4.

---

### Step 8: Create the Triggers (auto-run on git push)

You need four objects: a **TriggerBinding** (extracts fields from the webhook payload), a
**TriggerTemplate** (the PipelineRun to create), an **EventListener** (with a ServiceAccount that
can create PipelineRuns), and a **Route** to expose the listener.

**8a. ServiceAccount + RBAC for the EventListener** (`trigger-rbac.yaml`).
You need **two** bindings — a namespaced RoleBinding *and* a ClusterRoleBinding. The EventListener
watches `ClusterInterceptor`, which is cluster-scoped, so a RoleBinding alone is not enough:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pipeline-trigger-sa
  namespace: cicd
---
# namespaced: eventlisteners, triggerbindings, triggertemplates, creating pipelineruns in cicd
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: pipeline-trigger-binding
  namespace: cicd
subjects:
  - kind: ServiceAccount
    name: pipeline-trigger-sa
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: tekton-triggers-eventlistener-roles          # shipped by the operator
---
# cluster-scoped: clusterinterceptors, clustertriggerbindings
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: pipeline-trigger-clusterbinding
subjects:
  - kind: ServiceAccount
    name: pipeline-trigger-sa
    namespace: cicd                                  # required on a ClusterRoleBinding
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: tekton-triggers-eventlistener-clusterroles   # shipped by the operator
```

> Creating a `ClusterRoleBinding` needs cluster-admin — log in as `admin` for this one object.
> If those ClusterRole names differ on your version, find them with `oc get clusterrole | grep -i trigger`.

Verify before moving on — this is faster than reading listener logs:
```bash
oc auth can-i list clusterinterceptors --as=system:serviceaccount:cicd:pipeline-trigger-sa
# -> yes
```

**8b. TriggerBinding** (`trigger-binding.yaml`) — maps GitLab push payload to params:
```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerBinding
metadata:
  name: pipeline-app-binding
  namespace: cicd
spec:
  params:
    - name: git-url
      value: $(body.project.git_http_url)
    - name: git-revision
      value: $(body.checkout_sha)
```
> For GitHub payloads use `$(body.repository.clone_url)` and `$(body.after)` instead.
> `git_http_url` returns whatever scheme GitLab is configured with — here `https://`, which matches
> the `.gitconfig` credential context in `git-basic-auth`. If the two ever disagree, the clone
> prompts for a username.

**8c. TriggerTemplate** (`trigger-template.yaml`) — the PipelineRun to spawn. It **must bind both
workspaces**, including `git-auth`, or the run fails validation before any pod starts:
```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: TriggerTemplate
metadata:
  name: pipeline-app-template
  namespace: cicd
spec:
  params:
    - name: git-url
    - name: git-revision
      default: main
  resourcetemplates:
    - apiVersion: tekton.dev/v1
      kind: PipelineRun
      metadata:
        generateName: build-and-deploy-trig-
      spec:
        pipelineRef:
          name: build-and-deploy
        taskRunTemplate:
          serviceAccountName: pipeline
        params:
          - name: git-url
            value: $(tt.params.git-url)
          - name: git-revision
            value: $(tt.params.git-revision)
        workspaces:
          - name: shared-data
            volumeClaimTemplate:
              spec:
                accessModes: [ReadWriteOnce]
                resources:
                  requests:
                    storage: 1Gi
          - name: git-auth
            secret:
              secretName: git-basic-auth
```

**8d. EventListener** (`event-listener.yaml`):
```yaml
apiVersion: triggers.tekton.dev/v1beta1
kind: EventListener
metadata:
  name: pipeline-app-listener
  namespace: cicd
spec:
  serviceAccountName: pipeline-trigger-sa
  triggers:
    - name: gitlab-push
      bindings:
        - ref: pipeline-app-binding
      template:
        ref: pipeline-app-template
```

Apply everything:
```bash
oc apply -f trigger-rbac.yaml
oc apply -f trigger-binding.yaml
oc apply -f trigger-template.yaml
oc apply -f event-listener.yaml

# The operator creates an el-<name> Deployment + Service
oc get eventlistener
oc get svc | grep el-pipeline-app-listener
```

**8e. Expose the EventListener with a route:**
```bash
oc expose svc el-pipeline-app-listener
EL_ROUTE=$(oc get route el-pipeline-app-listener -o jsonpath='{.spec.host}')
echo "Webhook URL: http://${EL_ROUTE}"
```

---

### Step 9: Trigger a run through the EventListener

**9a. Verify the trigger chain with a direct POST.** Do this first — it exercises the
EventListener, TriggerBinding, TriggerTemplate, and RBAC without involving GitLab at all, so a
failure here is yours to fix:

```bash
EL_ROUTE=$(oc get route el-pipeline-app-listener -n cicd -o jsonpath='{.spec.host}')

curl -i -X POST "http://${EL_ROUTE}" \
  -H 'Content-Type: application/json' \
  -d '{"checkout_sha":"main","project":{"git_http_url":"https://git.ocp4.example.com/developer/pipeline-app.git"}}'

oc get pipelinerun -n cicd -w
```

A `202 Accepted` plus a new `build-and-deploy-trig-*` PipelineRun means the whole trigger chain is
correct.

**9b. Configure the GitLab webhook.** In the `pipeline-app` project on
`https://git.ocp4.example.com`: **Settings → Webhooks** →

- URL: `http://<EL_ROUTE>` — print it with the scheme included, GitLab rejects a bare hostname:
  ```bash
  oc get route el-pipeline-app-listener -n cicd -o jsonpath='{"http://"}{.spec.host}{"\n"}'
  ```
- Trigger: **Push events**
- (Lab) untick SSL verification
- **Add webhook**, then **Test → Push events**

> **If GitLab says "invalid URL".** GitLab blocks webhooks that resolve to the local network and
> reports it as a URL validation error. `apps.ocp4.example.com` resolves to an RFC1918 address, so
> the classroom GitLab rejects the EventListener route until an admin allows it. Confirm the
> diagnosis:
> ```bash
> getent hosts $(oc get route el-pipeline-app-listener -n cicd -o jsonpath='{.spec.host}')
> # 192.168.x.x / 10.x.x.x  -> local-network block
> ```
> The setup should already have enabled this (see `question10.md` Step 6). If not, from workstation:
> ```bash
> ssh 192.168.50.50            # the GitLab VM
> su -                         # classroom root password: redhat
> cat /etc/gitlab/initial_root_password       # GitLab admin account is 'root'
> gitlab-rake "gitlab:password:reset[root]"   # if that password no longer works
> ```
> Then, as `root` in the GitLab UI: **Admin Area → Settings → Network → Outbound requests** →
> enable *Allow requests to the local network from webhooks and integrations* (and the system-hooks
> equivalent) → **Save changes**. Re-add the webhook; it now saves and fires.

**9c. Confirm a real push triggers a run.** Edit `index.html` in the repo, commit, and push:

```bash
oc get pipelinerun -n cicd -w
```

A new `build-and-deploy-trig-*` run appears within a few seconds of the push, and the updated page
is served at `http://pipeline-app-cicd.apps.ocp4.example.com` once it completes.

---

### Step 10: Verify the application

```bash
oc get pods
oc get route pipeline-app
curl http://pipeline-app-cicd.apps.ocp4.example.com
# -> Hello, Pipelines!
```

---

## Success Criteria

- Project `cicd` contains a `Pipeline/build-and-deploy` with tasks
  `fetch-repository` → `build-image` → `deploy` (serialized with `runAfter`).
- The `shared-data` workspace, backed by a PVC (static or `volumeClaimTemplate`), is shared by
  the tasks, and `git-auth` supplies the `git-clone` `basic-auth` credentials.
- A manually started `PipelineRun` completes with all tasks `Succeeded`.
- A `TriggerTemplate`, `TriggerBinding`, and `EventListener` exist, with **both** a RoleBinding and
  a ClusterRoleBinding on `pipeline-trigger-sa`; the EventListener is exposed via a route, and a
  `git push` to the repository starts a new `PipelineRun` automatically.
- The app answers `Hello, Pipelines!` at `http://pipeline-app-cicd.apps.ocp4.example.com`.

---

## Key Commands Reference
```bash
# Pipelines / tasks
tkn task list -n openshift-pipelines
tkn task describe git-clone -n openshift-pipelines
tkn pipeline list
tkn pipeline describe build-and-deploy

# Start & watch runs
oc create -f pipelinerun.yaml
tkn pipeline start build-and-deploy \
  --workspace name=shared-data,claimName=shared-data-pvc \
  --workspace name=git-auth,secret=git-basic-auth --showlog
tkn pipelinerun list
tkn pipelinerun logs -f --last
oc get pipelinerun

# Permissions (pipeline SA)
oc policy add-role-to-user system:image-builder -z pipeline
oc policy add-role-to-user edit -z pipeline

# Git credentials
oc get secret git-basic-auth -n cicd

# Triggers
oc get eventlistener,triggertemplate,triggerbinding
oc get svc | grep el-
oc expose svc el-pipeline-app-listener
oc get route el-pipeline-app-listener -o jsonpath='{"http://"}{.spec.host}{"\n"}'
oc auth can-i list clusterinterceptors --as=system:serviceaccount:cicd:pipeline-trigger-sa
oc logs -f deploy/el-pipeline-app-listener -n cicd
oc rollout restart deploy/el-pipeline-app-listener -n cicd
```

---

## Common Issues and Troubleshooting

| Issue | Symptom | Fix |
|-------|---------|-----|
| **Malformed git URL** | `fatal: 'https//git...' does not appear to be a git repository`, preceded by `No such remote 'origin'` | The `://` is missing. git treated the string as a local path. Fix the `git-url` param **and** the pipeline's default |
| **Self-signed certificate** | `SSL certificate problem: self-signed certificate in certificate chain` | Set `SSL_VERIFY: "false"` on `fetch-repository`. Verify it landed — the log must show `-sslVerify=false` |
| **`SSL_VERIFY` has no effect** | log still shows `-sslVerify=true` | Wrong param name for your version, or the pipeline was not reapplied. Confirm with `tkn task describe git-clone -n openshift-pipelines` |
| **Private repo, no credentials** | `could not read Username for 'https://git...': No such device or address` | Bind the `git-basic-auth` secret to the task's `basic-auth` workspace (Step 4 + Step 5) |
| **Credentials present but still prompting** | log shows the files copied to `/tekton/home`, yet git still prompts | `USER_HOME` was overridden. **Leave it at its `/home/git` default** — that is where the step's `HOME` points. Overriding to `/tekton/home` moves the files away from git |
| **SA-linked secret ignored** | `oc secrets link pipeline git-credentials` done, annotation correct, still prompts | creds-init writes to `/tekton/home` but git runs with `HOME=/home/git`. Use the `basic-auth` workspace instead |
| **Repo path typo looks like an auth error** | `could not read Username` on a repo that does not exist | GitLab returns an auth challenge for missing repos. Verify with `curl -sk -o /dev/null -w '%{http_code}\n' -u user:pass https://git.ocp4.example.com/developer/pipeline-app.git/info/refs?service=git-upload-pack` (200 = reachable, 404 = wrong path) |
| **Dockerfile not found** | `ERROR: unable to find the Dockerfile, DOCKERFILE may have an incorrect location` | `DOCKERFILE` must match the filename in the repo — `./Dockerfile` here. Do not double-prefix it with `CONTEXT` |
| **Misspelled param silently ignored** | a param you set has no effect (e.g. `TLSVERIFY` vs `TLS_VERIFY`) | Read the real names off the cluster with `tkn task describe <task> -n openshift-pipelines` |
| **buildah push fails** | `build-image` task errors with auth/permission denied | `oc policy add-role-to-user system:image-builder -z pipeline` |
| **deploy task forbidden** | `oc new-app`/`apply` returns `Forbidden` | `oc policy add-role-to-user edit -z pipeline` |
| **Task not found** | `couldn't retrieve task "git-clone"` | Use `resolver: cluster` with `namespace: openshift-pipelines`, or `kind: ClusterTask` on older versions |
| **Workspace not bound** | PipelineRun stays `Pending`, "workspace not provided" | Every workspace the Pipeline declares must be bound — `shared-data` **and** `git-auth` — in the PipelineRun *and* the TriggerTemplate |
| **RWO multi-attach error** | Parallel tasks fail to mount the PVC | Serialize with `runAfter`, or request RWX storage |
| **EventListener 202 but no run** | POST accepted, no PipelineRun created | Check `oc logs deploy/el-pipeline-app-listener`; usually missing RBAC on `pipeline-trigger-sa`, or the TriggerTemplate omits the `git-auth` workspace |
| **`clusterinterceptors is forbidden`** | listener pod runs but log loops `cannot list resource "clusterinterceptors" ... at the cluster scope` | A RoleBinding cannot grant a cluster-scoped resource. Add the **ClusterRoleBinding** to `tekton-triggers-eventlistener-clusterroles` (Step 8a), then `oc rollout restart deploy/el-pipeline-app-listener` |
| **GitLab rejects the webhook URL** | "invalid URL" in the GitLab UI when saving the webhook | Either the scheme is missing (paste `http://<host>`, not the bare host), or GitLab blocks local-network webhooks. For the latter, log in as GitLab's `root` (password in `/etc/gitlab/initial_root_password` on the GitLab VM, or reset with `gitlab-rake "gitlab:password:reset[root]"`) and enable **Admin → Settings → Network → Outbound requests → allow local network** |
| **Binding value empty** | `git-url`/`git-revision` blank in the run | Payload path wrong — GitLab uses `body.project.git_http_url`, GitHub uses `body.repository.clone_url` |
| **Route unreachable** | Webhook test fails to connect | `oc expose svc el-pipeline-app-listener` and use the `el-` route host, not the app route |

---

## Notes / Version differences

- **ClusterTask vs cluster resolver:** OpenShift Pipelines is migrating away from `ClusterTask`. On
  older lab clusters use:
  ```yaml
  taskRef:
    name: git-clone
    kind: ClusterTask
  ```
  On newer ones use the `resolver: cluster` form shown above. Check with
  `oc get clustertask` — if it returns objects, `ClusterTask` still works.
- **API versions:** pipelines use `tekton.dev/v1`; triggers here use `triggers.tekton.dev/v1beta1`
  (some clusters still expose `v1alpha1` — `oc api-resources | grep triggers` to confirm).
- **Git auth mechanisms:** `git-clone` supports three, in descending order of reliability on this
  lab — the `basic-auth` workspace (used here), the `ssh-directory` workspace, and SA-linked
  secrets annotated `tekton.dev/git-0` (creds-init). Red Hat's own task description recommends
  `ssh-directory` over `basic-auth` where SSH keys are available.
- **`ssl-ca-directory` workspace:** instead of disabling verification, you can mount the lab CA.
  Bind a ConfigMap containing `ca-bundle.crt` to the task's `ssl-ca-directory` workspace; the
  filename comes from the `CRT_FILENAME` param (default `ca-bundle.crt`).
- **`tkn` is your friend under time pressure:** most param and workspace names you need are shown by
  `tkn task describe <name> -n openshift-pipelines`, so you don't have to hunt through docs.
