
```markdown
# Question 2: Optimize a Containerfile

## Question

Optimize, push, and use the Containerfile from `http://git.ocp4.example.com/developer/build.git` such that:

- An application exists in the **container-build* project and is available at `http://aap-name-container-build.apps.ocp4.example.com`
- An image built with the Containerfile can be used as a parent image to generate child images which allow overriding default content from `src/`
- An image built with the Containerfile has a maximum of **7 layers** and a maximum size of **256 MiB**

```

## Environment Setup

### Step 1: Create the Build Repository with Unoptimized Dockerfile

Create the project structure with an intentionally inefficient Dockerfile:

```bash
mkdir -p ~/build-unoptimized
cd ~/build-unoptimized
```

**Create unoptimized `Dockerfile` (Multiple RUN statements, separate LABELs):**

```bash
cat > Dockerfile <<'EOF'
FROM registry.access.redhat.com/ubi8/ubi-minimal:latest

LABEL version="1.0"
LABEL description="this is Dockerfile"
LABEL Red Hat Training <training@redhat.com>

USER root

RUN microdnf install -y python3
RUN microdnf clean all
RUN mkdir -p /app
RUN echo "Hello container!" > /app/index.html

ENV DOCROOT=/app

EXPOSE 8080

USER 1001

WORKDIR /app

CMD ["python3", "-m", "http.server", "8080"]
EOF
```

**Create `src/` directory with test content:**

```bash
mkdir src
cat > src/index.html <<'EOF'
<!DOCTYPE html>
<html>
<head>
  <title>Child Image Override</title>
</head>
<body>
  <h1>This content is from the child image src/ directory</h1>
  <p>If you see this, COPY worked correctly!</p>
</body>
</html>
EOF
```

---

### Step 2: Push to GitLab

**Initialize and commit:**

```bash
git init
git add .
git commit -m "Initial unoptimized Dockerfile"
```

* You can also create the repo first and clone it then push

**Create the repo in GitLab:**

1. Go to `https://git.ocp4.example.com`
2. Login as `developer` / `developer`
3. Click **"New project"** → **"Create blank project"**
4. Project name: **build**
5. **IMPORTANT:** Set visibility to **Public** or **Internal**
6. Click **"Create project"**

**Push the code:**

```bash
git checkout -b master
git remote add origin https://git.ocp4.example.com/developer/build.git
git push -u origin master
```
---

### Step 3: Verify Repository Structure

Go to `https://git.ocp4.example.com/developer/build` and verify you see:

```
build/
├── Dockerfile
├── src/
│   └── index.html
```

**Setup complete.** The unoptimized Dockerfile is ready for the solution phase.

---
