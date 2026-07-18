## Solution (Timed Exercise — 45 minutes)

This is the longest task on the exam. Work top-to-bottom and **verify each object renders before
applying it**. The flow is: permissions → workspace PVC → tasks → pipeline → manual PipelineRun →
triggers → webhook → verify.

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

### Step 4: Create the Pipeline

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
      default: http://git.ocp4.example.com/developer/pipeline-app.git
    - name: git-revision
      type: string
      default: main
    - name: image
      type: string
      default: image-registry.openshift-image-registry.svc:5000/cicd/pipeline-app:latest
  workspaces:
    - name: shared-data          # <-- the single shared workspace
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
      params:
        - name: URL
          value: $(params.git-url)
        - name: REVISION
          value: $(params.git-revision)
        - name: DELETE_EXISTING
          value: "true"

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
          value: ./Containerfile
        - name: TLSVERIFY
          value: "false"

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
> `--tls-verify` needed for the pull inside the cluster.

---

### Step 5: (Optional) Inspect the shipped task params

If you are unsure of a task's exact param names (they differ slightly by version), read them:
```bash
tkn task describe git-clone -n openshift-pipelines
tkn task describe buildah -n openshift-pipelines
oc get task buildah -n openshift-pipelines -o yaml | less
```

---

### Step 6: Start a PipelineRun manually and confirm it succeeds

**Option A — with `tkn` (auto-creates a `volumeClaimTemplate` workspace):**
```bash
tkn pipeline start build-and-deploy \
  --param git-url=http://git.ocp4.example.com/developer/pipeline-app.git \
  --param git-revision=main \
  --workspace name=shared-data,volumeClaimTemplateFile=- <<'EOF'
spec:
  accessModes: [ReadWriteOnce]
  resources:
    requests:
      storage: 1Gi
EOF

# Or, simpler, bind to the static PVC from Step 3:
tkn pipeline start build-and-deploy \
  --workspace name=shared-data,claimName=shared-data-pvc \
  --showlog
```

**Option B — declarative PipelineRun YAML (`pipelinerun.yaml`):**
```yaml
apiVersion: tekton.dev/v1
kind: PipelineRun
metadata:
  generateName: build-and-deploy-run-
  namespace: cicd
spec:
  pipelineRef:
    name: build-and-deploy
  workspaces:
    - name: shared-data
      volumeClaimTemplate:            # fresh PVC per run
        spec:
          accessModes: [ReadWriteOnce]
          resources:
            requests:
              storage: 1Gi
      # --- or bind the static PVC instead ---
      # persistentVolumeClaim:
      #   claimName: shared-data-pvc
```
```bash
oc create -f pipelinerun.yaml
```

**Watch it:**
```bash
tkn pipelinerun logs -f --last
tkn pipelinerun list
oc get pipelinerun
```
All three tasks must reach `Succeeded`.

---

### Step 7: Create the Triggers (auto-run on git push)

You need four objects: a **TriggerBinding** (extracts fields from the webhook payload), a
**TriggerTemplate** (the PipelineRun to create), an **EventListener** (with a ServiceAccount that
can create PipelineRuns), and a **Route** to expose the listener.

**7a. ServiceAccount + RBAC for the EventListener** (`trigger-rbac.yaml`):
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: pipeline-trigger-sa
  namespace: cicd
---
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
  name: tekton-triggers-eventlistener-roles     # shipped by the operator
```
> If that ClusterRole name is not present, use `oc get clusterrole | grep triggers` to find the
> `...eventlistener-roles` and `...eventlistener-clusterroles` names for your version and bind both.

**7b. TriggerBinding** (`trigger-binding.yaml`) — maps GitLab push payload to params:
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

**7c. TriggerTemplate** (`trigger-template.yaml`) — the PipelineRun to spawn:
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
```

**7d. EventListener** (`event-listener.yaml`):
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

**7e. Expose the EventListener with a route:**
```bash
oc expose svc el-pipeline-app-listener
EL_ROUTE=$(oc get route el-pipeline-app-listener -o jsonpath='{.spec.host}')
echo "Webhook URL: http://${EL_ROUTE}"
```

---

### Step 8: Configure the GitLab webhook

In the `pipeline-app` project on `http://git.ocp4.example.com`:
**Settings → Webhooks** →
- URL: `http://<EL_ROUTE>`   (the value printed above)
- Trigger: **Push events**
- (Lab) untick SSL verification
- **Add webhook**, then **Test → Push events**

You can also test locally without GitLab:
```bash
curl -X POST http://${EL_ROUTE} \
  -H 'Content-Type: application/json' \
  -d '{"checkout_sha":"main","project":{"git_http_url":"http://git.ocp4.example.com/developer/pipeline-app.git"}}'
```

A new PipelineRun should appear:
```bash
oc get pipelinerun -w
```

---

### Step 9: Verify the application

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
- A single workspace `shared-data`, backed by a PVC (static or `volumeClaimTemplate`), is shared by
  the tasks.
- A manually started `PipelineRun` completes with all tasks `Succeeded`.
- A `TriggerTemplate`, `TriggerBinding`, and `EventListener` exist; the EventListener is exposed via
  a route and a `git push` (or a test POST) starts a new `PipelineRun` automatically.
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
tkn pipeline start build-and-deploy --workspace name=shared-data,claimName=shared-data-pvc --showlog
tkn pipelinerun list
tkn pipelinerun logs -f --last
oc get pipelinerun

# Permissions (pipeline SA)
oc policy add-role-to-user system:image-builder -z pipeline
oc policy add-role-to-user edit -z pipeline

# Triggers
oc get eventlistener,triggertemplate,triggerbinding
oc get svc | grep el-
oc expose svc el-pipeline-app-listener
oc get route el-pipeline-app-listener -o jsonpath='{.spec.host}'
```

---

## Common Issues and Troubleshooting

| Issue | Symptom | Fix |
|-------|---------|-----|
| **buildah push fails** | `build-image` task errors with auth/permission denied | `oc policy add-role-to-user system:image-builder -z pipeline` |
| **deploy task forbidden** | `oc new-app`/`apply` returns `Forbidden` | `oc policy add-role-to-user edit -z pipeline` |
| **Task not found** | `couldn't retrieve task "git-clone"` | Use `resolver: cluster` with `namespace: openshift-pipelines`, or `kind: ClusterTask` on older versions |
| **Workspace not bound** | PipelineRun stays `Pending`, "workspace not provided" | Provide `--workspace` (tkn) or a `workspaces:` entry (YAML) for `shared-data` |
| **RWO multi-attach error** | Parallel tasks fail to mount the PVC | Serialize with `runAfter`, or request RWX storage |
| **EventListener 202 but no run** | POST accepted, no PipelineRun created | Check `oc logs deploy/el-pipeline-app-listener`; usually missing RBAC on `pipeline-trigger-sa` |
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
- **`tkn` is your friend under time pressure:** most param names you need are shown by
  `tkn task describe <name> -n openshift-pipelines`, so you don't have to hunt through docs.
