# Question 5: Customize Build Process with Post-Commit Hook

## Question

A Python 3 application named **blog** has been deployed for you in a project named **octane**.

The application includes a script named: **mailer.py**

Customize the application such that the following statements are true:

- The application is running and available at `http://blog-octane.apps.ocp4.example.com`
- After a build finishes, the above-mentioned script is automatically executed
- The most recent build of the application succeeded and triggered the script to run
- Future rebuilds of the application will trigger the script to run
- The original Git repository used to create the application has not been modified by you

***The mail doesnt have to send just make sure that the script executes after the build**

---

## Environment Setup

### Step 1: Create Empty Repository in GitLab

**Create the repo in GitLab UI:**

1. Go to `https://git.ocp4.example.com`
2. Login as `developer` / `developer`
3. Click **"New project"** → **"Create blank project"**
4. Project name: **blog**
5. **IMPORTANT:** 
   - Set visibility to **Public** or **Internal**
   - **Check** "Initialize repository with a README"
6. Click **"Create project"**

---

### Step 2: Clone the Empty Repository
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
    return 'OK'

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

**Create the mailer.py script:**
```bash
cat > mailer.py <<'EOF'
#!/usr/bin/env python3
import os
import subprocess
from datetime import datetime

def send_mail():
    """Send email notification after build completes"""
    build_name = os.environ.get('OPENSHIFT_BUILD_NAME', 'unknown-build')
    build_namespace = os.environ.get('OPENSHIFT_BUILD_NAMESPACE', 'unknown-namespace')
    timestamp = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
    
    subject = f"Build {build_name} completed"
    body = f"""
Build Notification
==================
Build Name: {build_name}
Namespace: {build_namespace}
Timestamp: {timestamp}
Status: SUCCESS

This email was sent automatically by the post-commit build hook.
"""
    
    # Send email using mail command
    try:
        # Write message to file
        with open('/tmp/mail_message.txt', 'w') as f:
            f.write(body)
        
        # Send mail
        mail_cmd = f'mail -s "{subject}" capnhook < /tmp/mail_message.txt'
        result = subprocess.run(mail_cmd, shell=True, capture_output=True, text=True)
        
        print(f"Email sent to capnhook user")
        print(f"Subject: {subject}")
        print(f"Return code: {result.returncode}")
        
        if result.returncode != 0:
            print(f"Warning: mail command returned non-zero exit code")
            print(f"Stderr: {result.stderr}")
    except Exception as e:
        print(f"Error sending email: {e}")

if __name__ == '__main__':
    send_mail()
    print("mailer.py script executed successfully")
EOF

chmod +x mailer.py
```

---

### Step 4: Commit and Push Initial Setup
```bash
git add .
git commit -m "Initial blog application with mailer.py script"
git push origin master
```

**Verify in GitLab:**

Go to `https://git.ocp4.example.com/developer/blog` and verify you see:
```
blog/
├── README.md
├── app.py
├── requirements.txt
└── mailer.py
```

---

### Step 5: Create the octane Project and Deploy Initial Application
```bash
# Create project
oc new-project octane

# Deploy the blog application
oc new-app --name=blog --strategy=source https://git.ocp4.example.com/developer/blog.git#master

# Watch the build
oc logs -f bc/blog

# Wait for deployment
oc get pods

# Expose the service
oc expose svc/blog
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

**Setup complete.** The blog application is deployed and ready for build hook customization.

---

