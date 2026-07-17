```markdown
# Question 1: Deploy an Application with S2I (Broken Code Fix)

## Question

Deploy an application on OpenShift using the source code from `http://git.ocp4.example.com/developer/pastebin.git` that meets the following requirements:

- The application is part of a project named: **crimson**
- The application is named: **pastebin**
- Once deployed, the application is running and available at `http://pastebin-crimson.apps.ocp4.example.com`
- You have used the application to create and save a pastebin entry containing the text:
  
  This is an OpenShift Demo!
  
- Note that application dependencies are available from the registry located at:
  
  http://nexus-infra.apps.ocp4.example.com/repository/npm/
  
  You will need to pass that to the build environment via the `npm_config_registry` parameter.

```

## Environment Setup

### Step 1: Create the Broken Pastebin Repo

Create the project structure with intentionally broken `package.json` from the env_code directory.

### Step 2: Push to GitLab

**Initialize and commit:**
```bash
git init
git add .
git commit -m "Initial commit"
```

**Create the repo in GitLab:**
1. Go to `https://git.ocp4.example.com`
2. Login as `developer` / `developer`
3. Click **"New project"** then **"Create blank project"**
4. Project name: **pastebin**
5. **IMPORTANT:** Set visibility to **Public** or **Internal** (NOT Private)
6. Click **"Create project"**

**Push the code:**
```bash
# Use master branch (OpenShift default in training env) or push to main 
git checkout -b master

# Add remote
git remote add origin https://git.ocp4.example.com/developer/pastebin.git

# Push
git push -u origin master
```
* You can also just clone the repo and push to the main branch 

### Step 3: Verify the Broken Code

Confirm the `package.json` is invalid:

```bash
python -m json.tool package.json
```

**Expected output:**
```
Expecting ',' delimiter: line 3 column 3 (char 44)
```

**Setup complete.** The broken code is ready for the solution phase.