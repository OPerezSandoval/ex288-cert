# Question 6: Implement Health Monitoring with Liveness Probe

## Question

An application named **blogger** has been deployed for you in a project named **blogger-app**.

The application is backed by a single running container. Implement a liveness probe for this container that meets the following requirements:

- The probe monitors liveness by performing a **TCP socket check on port 8080**
- The probe has an **initial delay of 10 seconds** and a **timeout of 30 seconds**
- Your changes can survive a rebuild

---

## Environment Setup

### Step 1: Create Empty Repository in GitLab

**Create the repo in GitLab UI:**

1. Go to `https://git.ocp4.example.com`
2. Login as `developer` / `developer`
3. Click **"New project"** → **"Create blank project"**
4. Project name: **blogger**
5. **IMPORTANT:** 
   - Set visibility to **Public** or **Internal**
   - **Check** "Initialize repository with a README"
6. Click **"Create project"**

---

### Step 2: Clone the Repository
```bash
cd ~
git clone https://git.ocp4.example.com/developer/blog.git
cd blog
```

---

### Step 3: Create Python Application Files

**Create the main application file:**
```bash
cat > app.py <<'EOF'
from flask import Flask
import os

app = Flask(__name__)

@app.route('/')
def hello():
    return '<h1>Blog Application</h1><p>This is a Python Flask blog app.</p>'

@app.route('/health')
def health():
    return 'OK', 200

if __name__ == '__main__':
    port = int(os.environ.get('PORT', 8080))
    app.run(host='0.0.0.0', port=port)
EOF
```

**Create requirements.txt:**
```bash
cat > requirements.txt <<'EOF'
Flask==2.3.0
Werkzeug==2.3.0
EOF
```

---

### Step 4: Commit and Push
```bash
git add .
git commit -m "Initial blog application"
git push origin master
```

---

### Step 5: Create Project and Deploy Application
```bash
# Create project
oc new-project blogger-app

# Deploy the blog application
oc new-app --name=blogger --strategy=source https://git.ocp4.example.com/developer/blogger.git#main 

# Watch the build
oc logs -f bc/blogger

# Wait for deployment
oc get pods

# Expose the service
oc expose svc/blogger
```

**Verify the application is running:**
```bash
curl http://blog-octane.apps.ocp4.example.com
```

**Expected output:**
```html
<h1>Blog Application</h1><p>This is a Python Flask blog app.</p>
```

---

**Setup complete.** The blog application is deployed and ready for liveness probe configuration.

---