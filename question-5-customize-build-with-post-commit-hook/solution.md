## Solution (Timed Exercise - 10 minutes)

### Step 1: Switch to the octane Project
```bash
oc project octane
```

---

### Step 2: Identify the Python Path in the Builder Image

The post-commit hook needs the correct Python path. Check which Python is available:
```bash
# Start a test build and check logs
oc start-build blog

# Or check the builder image
oc get bc/blog -o yaml | grep "image:"
```

**For Python 3.9 S2I images, Python is typically at:**
- `/opt/app-root/bin/python3`
- `/usr/bin/python3`

---

### Step 3: Set the Post-Commit Build Hook
```bash
oc set build-hook bc/blog \
  --post-commit \
  --command -- /opt/app-root/bin/python3 mailer.py
```

**Alternative if `/opt/app-root/bin/python3` doesn't work:**
```bash
oc set build-hook bc/blog \
  --post-commit \
  --command -- python3 mailer.py
```

**Or use --script (simpler):**
```bash
oc set build-hook bc/blog \
  --post-commit \
  --script="python3 mailer.py"
```

---

### Step 4: Verify the Post-Commit Hook was Created
```bash
oc describe bc/blog | grep -A 5 "Post Commit"
```

**Expected output:**
```
Post Commit:
  Command:
    /opt/app-root/bin/python3
    mailer.py
```

**Or:**
```bash
oc get bc/blog -o yaml | grep -A 10 postCommit
```

---

### Step 5: Trigger a New Build to Test the Hook
```bash
oc start-build blog --follow
```

**Watch for the mailer.py execution in the logs:**
```
Post Commit Hook execution output:
Email sent to capnhook user
Subject: Build blog-2 completed
Return code: 0
mailer.py script executed successfully
```

**Note:** You may see warnings like "network unreachable" - these can be safely ignored as mentioned in the question.

---

### Step 6: Verify Build Completed Successfully
```bash
oc get builds
```

**Expected output:**
```
NAME      TYPE     FROM          STATUS     STARTED          DURATION
blog-1    Source   Git@master    Complete   10 minutes ago   2m30s
blog-2    Source   Git@master    Complete   2 minutes ago    2m15s
```

---

### Step 7: Check Application is Still Running
```bash
oc get pods

curl http://blog-octane.apps.ocp4.example.com
```

---

### Step 8: Verify Email was Sent 

```
In the logs of the build you should see that it ran the script 
```

---

### Step 9: Verify Future Builds Will Trigger the Script
```bash
# Trigger another build
oc start-build blog

# Check that it also executes mailer.py
oc logs -f bc/blog

```

---

## Success Criteria

- Project `octane` exists
- Application `blog` is running
- Route `blog-octane.apps.ocp4.example.com` is accessible
- Post-commit build hook is set on BuildConfig `blog`
- Most recent build (blog-2 or later) completed successfully
- mailer.py script executed after the build
- Future builds will trigger the post-commit hook
- Original Git repository was NOT modified

---

## Key Commands Reference
```bash
# Set post-commit build hook (command style)
oc set build-hook bc/<name> --post-commit --command -- <command> <args>

# Set post-commit build hook (script style)
oc set build-hook bc/<name> --post-commit --script="<script>"

# Verify build hook
oc describe bc/<name> | grep -A 5 "Post Commit"

# View BuildConfig YAML
oc get bc/<name> -o yaml

# Trigger build
oc start-build <name>

# Watch build logs
oc logs -f bc/<name>

# Check build status
oc get builds

# Remove build hook
oc set build-hook bc/<name> --post-commit --remove

```

---

## Understanding Build Hooks

### What are Build Hooks?

Build hooks allow you to run commands at specific points in the build process:

- **Post-commit hook**: Runs AFTER the build completes successfully
- Useful for: notifications, testing, artifact uploads, cleanup

### Hook Types:

1. **Command style** (array of strings):
```bash
oc set build-hook bc/blog --post-commit --command -- python3 mailer.py
```

2. **Script style** (single shell command):
```bash
oc set build-hook bc/blog --post-commit --script="python3 mailer.py"
```

### When to Use Build Hooks:

- Send notifications after successful builds
- Run tests on the built image
- Upload artifacts to external storage
- Tag images in a registry
- Trigger downstream builds

---

## Common Issues and Troubleshooting

| Issue | Symptom | Fix |
|-------|---------|-----|
| **Hook doesn't execute** | No script output in logs | Verify hook is set: `oc describe bc/blog` |
| **Wrong Python path** | Command not found error | Use `/opt/app-root/bin/python3` or `python3` |
| **Script not executable** | Permission denied | Ensure `chmod +x mailer.py` before commit |
| **Build fails** | Build doesn't complete | Check build logs: `oc logs -f bc/blog` |
| **Hook removed accidentally** | No longer executing | Re-apply: `oc set build-hook bc/blog --post-commit --script="python3 mailer.py"` |

**If post-commit hook doesn't run:**
```bash
# Check if hook exists
oc get bc/blog -o yaml | grep -A 10 postCommit

# Verify script is in the repo
oc start-build blog
oc logs blog-X-build | grep mailer

# Check Python path
oc rsh <build-pod> which python3

# Re-apply hook
oc set build-hook bc/blog --post-commit --script="python3 mailer.py"
```

---

## Build Hook Script Best Practices

### 1. Use Environment Variables

OpenShift provides useful variables in build hooks:
```bash
OPENSHIFT_BUILD_NAME       # e.g., "blog-2"
OPENSHIFT_BUILD_NAMESPACE  # e.g., "octane"
OPENSHIFT_BUILD_SOURCE     # Git URL
OPENSHIFT_BUILD_COMMIT     # Git commit SHA
```

### 2. Handle Errors Gracefully
```python
try:
    # Your code
except Exception as e:
    print(f"Error: {e}")
    # Don't fail the build
```

### 3. Keep Scripts Short

- Long-running hooks delay deployments
- For complex tasks, trigger external jobs instead

---

## Additional Practice

**Try these variations:**

### 1. Add Multiple Commands in Hook
```bash
oc set build-hook bc/blog --post-commit --script="python3 mailer.py && echo 'Done'"
```

### 2. Remove Build Hook
```bash
oc set build-hook bc/blog --post-commit --remove
```

---

## Cleanup (After Exercise)
```bash
oc delete project octane
cd ~
rm -rf blog
```
---