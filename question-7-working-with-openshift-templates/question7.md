# Question 7: Working with OpenShift Templates

## Question

Using the PHP+MySQL template from `/materials directory`, deploy an application that meets the following requirements:

- The application is part of a project named: **indy**
- The template is named: **ex288-php-mysql**
- The template has the label: **template: php-app**
- The application displays a customized greeting at `http://php-app-indy.apps.ocp4.example.com`
- The greeting displays: **"Namaste Architects!"** (instead of the default "Bonjour Engineers!")
- All required parameters in the template must be properly set
- The application successfully deploys from the template

---

## Environment Setup

### Step 1: Create the PHP Application Repository in GitLab

**Create the repo in GitLab UI:**

1. Go to `https://git.ocp4.example.com`
2. Login as `developer` / `d3v3lop3r`
3. Click **"New project"** → **"Create blank project"**
4. Project name: **php-greeting-app**
5. **IMPORTANT:** 
   - Set visibility to **Public** or **Internal**
6. Click **"Create project"**

---

### Step 2: Clone the Repository
```bash
cd ~
git clone https://git.ocp4.example.com/developer/php-greeting-app.git
cd php-greeting-app
```

---

### Step 3: Create PHP Application Files

**Create the main PHP application:**
```bash
cat > index.php <<'EOF'
<?php
$message = getenv('HELLO_MESSAGE') ?: 'Hello';
$audience = getenv('HELLO_AUDIENCE') ?: 'World';
?>
<!DOCTYPE html>
<html>
<head>
    <title>Greeting Application</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            text-align: center;
            padding: 50px;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
        }
        h1 {
            font-size: 4em;
            margin: 20px 0;
        }
    </style>
</head>
<body>
    <h1><?php echo htmlspecialchars($message) . ' ' . htmlspecialchars($audience); ?>!</h1>
    <p>This greeting is brought to you by OpenShift Templates</p>
</body>
</html>
EOF
```

---

### Step 4: Commit and Push
```bash
git add .
git commit -m "Initial PHP greeting application"
git push origin main 

**Verify in GitLab:**

Go to `https://git.ocp4.example.com/developer/php-greeting-app` and verify you see:
```
php-greeting-app/
├── README.md
└── index.php
```

---

### Step 5: Create the Template File (For Materials Server)

This simulates the template that would be downloaded from `/materials directory`.

**Create a materials directory:**
```bash
mkdir -p ~/materials
cd ~/materials
```

**Create the template file with default (incorrect) values:**
```bash
cat > php-app.yaml <<'EOF'
apiVersion: template.openshift.io/v1
kind: Template
labels:
  template: "wrong-label"
  app: "php-application"
metadata:
  name: ex288-php-mysql
  labels:
    template: "wrong-label"
  annotations:
    openshift.io/display-name: "PHP + MySQL (Persistent)"
    description: "A simple two-tier application to demonstrate a CI/CD pipeline from scratch."
    tags: "quickstart,php"
    iconClass: "icon-php"
    openshift.io/long-description: "This template defines resources for a LAMP application, including a build configuration, application deployment configuration, and database deployment configuration with persistent storage."
parameters:
  - name: NAME
    displayName: Name
    description: The name assigned to all of the frontend objects
    required: false
    value: lamp-app
  - name: HELLO_AUDIENCE
    displayName: Greeting Audience
    description: Who should we greet?
    required: false
    value: Engineers
  - name: HELLO_MESSAGE
    displayName: Greeting Message
    description: The greeting message
    required: false
    value: Bonjour
  - name: NAMESPACE
    displayName: Namespace
    description: The OpenShift Namespace where the ImageStream resides
    required: false
    value: openshift
  - name: APPLICATION_DOMAIN
    displayName: Application Hostname
    description: The exposed hostname that will route to the PHP service
    required: false
  - name: SOURCE_REPOSITORY_URL
    displayName: Git Repository URL
    description: The URL of the repository with your application source code
    required: false
    value: https://git.ocp4.example.com/developer/php-greeting-app.git
  - name: SOURCE_REPOSITORY_REF
    displayName: Git Reference
    description: Set this to a branch name, tag or other ref of your repository
    value: main
  - name: CONTEXT_DIR
    displayName: Context Directory
    description: Set this to the relative path to your project if it is not in the root
  - name: PHP_VERSION
    displayName: PHP Version
    description: Version of PHP to be used
    required: false
    value: "7.4-ubi8"
objects:
  - apiVersion: v1
    kind: Service
    metadata:
      name: ${NAME}
      annotations:
        description: Exposes and load balances the application pods
    spec:
      ports:
        - name: web
          port: 8080
          targetPort: 8080
      selector:
        name: ${NAME}
  - apiVersion: route.openshift.io/v1
    kind: Route
    metadata:
      name: ${NAME}
    spec:
      host: ${APPLICATION_DOMAIN}
      to:
        kind: Service
        name: ${NAME}
  - apiVersion: image.openshift.io/v1
    kind: ImageStream
    metadata:
      name: ${NAME}
      annotations:
        description: Keeps track of changes in the application image
  - apiVersion: build.openshift.io/v1
    kind: BuildConfig
    metadata:
      name: ${NAME}
      annotations:
        description: Defines how to build the application
        template.alpha.openshift.io/wait-for-ready: "true"
    spec:
      source:
        type: Git
        git:
          uri: ${SOURCE_REPOSITORY_URL}
          ref: ${SOURCE_REPOSITORY_REF}
        contextDir: ${CONTEXT_DIR}
      strategy:
        type: Source
        sourceStrategy:
          from:
            kind: ImageStreamTag
            namespace: ${NAMESPACE}
            name: php:${PHP_VERSION}
          env:
            - name: HELLO_MESSAGE
              value: ${HELLO_MESSAGE}
            - name: HELLO_AUDIENCE
              value: ${HELLO_AUDIENCE}
      output:
        to:
          kind: ImageStreamTag
          name: ${NAME}:latest
      triggers:
        - type: ImageChange
        - type: ConfigChange
  - apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: ${NAME}
      annotations:
        description: Defines how to deploy the application
        template.alpha.openshift.io/wait-for-ready: "true"
    spec:
      replicas: 2
      selector:
        matchLabels:
          name: ${NAME}
      template:
        metadata:
          labels:
            name: ${NAME}
        spec:
          containers:
            - name: php-app
              image: image-registry.openshift-image-registry.svc:5000/indy/${NAME}:latest
              ports:
                - containerPort: 8080
              env:
                - name: HELLO_MESSAGE
                  value: ${HELLO_MESSAGE}
                - name: HELLO_AUDIENCE
                  value: ${HELLO_AUDIENCE}
EOF
```

---

### Step 6: Make Template Available (Simulate Materials Server)

For exam simulation, we'll serve this file locally:
```bash
Just keep it in ~/materials for students to copy
# Students will use: wget file:///home/student/materials/php-app.yaml
```

---

**Setup complete.** The PHP application code is in GitLab and the template file is ready for modification.

---