
---
# Question 3: Customize S2I Builder Image Scripts

In concert with the source code from `http://git.ocp4.example.com/developer/oxy.git`, customize the behavior of the existing `httpd-24` builder image scripts to deploy an application that meets the following requirements:

- **Build the S2I image in the `s2i-builds` namespace**
- **Deploy the application in the `tocin` namespace, consuming the image from `s2i-builds`**
- The application is named: **oxy**
- During the build of the image, as part of the assemble script, `/tmp/src/*.html` files are copied into `./`
- Once deployed, the application is running and available at `http://oxy-tocin.apps.ocp4.example.com`, displaying the following text:
  ```
  This is the application oxy. If you see this its working.  
  ```
- Browsing to `http://oxy-tocin.apps.ocp4.example.com/info.html` displays an information page with the date when the application was built (YYYY-mm-dd format), as well as the text:
  ```
  Your info.html is working if you see this. 
  ```

---

## Environment Setup

### Step 1: Create Empty Repository in GitLab First

**Create the repo in GitLab UI:**

1. Go to `https://git.ocp4.example.com`
2. Login as `developer` / `developer`
3. Click **"New project"** → **"Create blank project"**
4. Project name: **oxy**
5. **IMPORTANT:** 
   - Set visibility to **Public** or **Internal**
   - **Check** "Initialize repository with a README"
6. Click **"Create project"**

---

### Step 2: Clone the Empty Repository

```bash
cd ~
git clone https://git.ocp4.example.com/developer/oxy.git
cd oxy
```

You should see:
```
oxy/
└── README.md
```

---

### Step 3: Create Application Files

**Create the main HTML file:**

```bash
cat > index.html <<'EOF'
<!DOCTYPE html>
<html>
<head>
  <title>Oxy Application</title>
</head>
<body>
  <h1>Amor vincit omnia</h1>
</body>
</html>
EOF
```

---

### Step 4: Create S2I Scripts Directory

**Create S2I scripts directory:**

```bash
mkdir -p .s2i/bin
```

**Create a basic (unmodified) assemble script:**

```bash
cat > .s2i/bin/assemble <<'EOF'
#!/bin/bash
# This is the default assemble script
# It does NOT copy HTML files or generate info.html yet

echo "Running default assemble script..."

# Source the default assemble from the builder image
if [ -f /usr/libexec/s2i/assemble ]; then
  /usr/libexec/s2i/assemble
fi
EOF

chmod +x .s2i/bin/assemble
```

---

### Step 5: Commit and Push Initial Setup

```bash
git add .
git commit -m "Initial oxy app without custom assemble modifications"
git push origin main
```

**Verify in GitLab:**

Go to `https://git.ocp4.example.com/developer/oxy` and verify you see:

```
oxy/
├── README.md
├── index.html
└── .s2i/
    └── bin/
        └── assemble
```

---

### Step 6: Create the Build Namespace

```bash
oc new-project s2i-builds
```

This namespace will hold the S2I image build.

---

**Setup complete.** The basic S2I structure is ready for customization, and the build namespace exists.

