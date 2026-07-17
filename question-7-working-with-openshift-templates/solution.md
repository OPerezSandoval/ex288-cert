## Solution (Timed Exercise - 20 minutes)

### Step 1: Create the Project
```bash
oc new-project indy
```

---

### Step 2: Download the Template
```bash
cd ~
wget file:///home/student/materials/php-app.yaml #this is may be a link provided by redhat 
```

**If the materials server isn't available, copy from setup:**
```bash
cp ~/materials/php-app.yaml ~/php-app.yaml
cd ~
```

---

### Step 3: Verify Template Before Modification
```bash
cat php-app.yaml | head -20
```

You should see:
- Template name: `ex288-php-mysql`
- Labels: `template: incorrect-label` (WRONG - needs to be `php-app`)
- Parameters with default values that need changing

---

### Step 4: Check if Template Already Exists
```bash
oc get template
```

**If template exists from previous attempts:**
```bash
oc delete template ex288-php-mysql
```

---

### Step 5: Apply the Template (First Attempt - Will Need Fixes)
```bash
oc apply -f php-app.yaml
```

**Verify template was created:**
```bash
oc get template ex288-php-mysql
```

---

### Step 6: Check Template Labels
```bash
oc get template --show-labels
```

**Expected output:**
```
NAME               LABELS
ex288-php-mysql    template=incorrect-label
```

**Problem:** Label says `incorrect-label` but should be `php-app` dont forget to add under the metadata labels

---

### Step 7: Fix Template Labels

**Edit the template file:**
```bash
vi php-app.yaml
```

**Change line 4:**
```yaml
# Before:
labels:
  template: incorrect-label

# After:
labels:
  template: php-app
```

**Also ensure metadata labels match:**
```yaml
metadata:
  labels:
    template: php-app
  name: ex288-php-template
```

Save with `:wq`

---

### Step 8: Fix Required Parameters

**Still in `php-app.yaml`, find the parameters section and make these changes:**
```yaml
parameters:
  - name: NAME
    displayName: Name
    description: The name assigned to all of the frontend objects
    required: true          # Change from false to true
    value: lamp-app
    
  - name: HELLO_AUDIENCE
    displayName: Greeting Audience
    description: Who should we greet?
    required: true          # Change from false to true
    value: Engineers        # Will override this when deploying
    
  - name: HELLO_MESSAGE
    displayName: Greeting Message
    description: The greeting message
    required: true          # Change from false to true
    value: Bonjour          # Will override this when deploying
    
  - name: NAMESPACE
    displayName: Namespace
    description: The OpenShift Namespace where the ImageStream resides
    required: true          # Change from false to true
    value: openshift
    
  - name: APPLICATION_DOMAIN
    displayName: Application Hostname
    description: The exposed hostname that will route to the PHP service
    required: true          # Change from false to true
    
  - name: SOURCE_REPOSITORY_URL
    displayName: Git Repository URL
    description: The URL of the repository with your application source code
    required: true          # Change from false to true
    value: https://git.ocp4.example.com/developer/php-greeting-app.git
    
  - name: PHP_VERSION
    displayName: PHP Version
    description: Version of PHP to be used
    required: true          # Change from false to true
    value: "7.4"
```

**Key changes:**
- All `required: false` → `required: true`
- `SOURCE_REPOSITORY_URL` value changed from `CHANGE_THIS_URL` to actual Git URL

Save with `:wq`

---

### Step 9: Delete and Re-apply Template
```bash
oc delete template ex288-php-mysql
oc apply -f php-app.yaml
```

---

### Step 10: Verify Template Labels
```bash
oc get template --show-labels
```

**Expected output:**
```
NAME               LABELS
ex288-php-mysql    template=php-app
```

**Success!** Label is now correct.

---

### Step 11: Deploy Application from Template
```bash
oc new-app ex288-php-app-template \
  -p NAME=php-app \
  -p HELLO_MESSAGE=Namaste \
  -p HELLO_AUDIENCE=Architects
```

```bash
apiVersion: template.openshift.io/v1
kind: Template
labels:
  template: "php-app"
  app: "php-application"
metadata:
  name: ex288-php-template
  labels:
    template: "php-app"
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
    required: true
    value: lamp-app
  - name: HELLO_AUDIENCE
    displayName: Greeting Audience
    description: Who should we greet?
    required: true
    value: Engineers
  - name: HELLO_MESSAGE
    displayName: Greeting Message
    description: The greeting message
    required: true
    value: Bonjour
  - name: NAMESPACE
    displayName: Namespace
    description: The OpenShift Namespace where the ImageStream resides
    required: true
    value: openshift
  - name: APPLICATION_DOMAIN
    displayName: Application Hostname
    description: The exposed hostname that will route to the PHP service
    required: true
  - name: SOURCE_REPOSITORY_URL
    displayName: Git Repository URL
    description: The URL of the repository with your application source code
    required: true
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
    required: true
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
```



**What this does:**
- Deploys the `ex288-php-app-template` template
- Sets `NAME` to `php-app`
- Sets custom hostname
- Overrides `HELLO_MESSAGE` from "Bonjour" to "Namaste"
- Overrides `HELLO_AUDIENCE` from "Engineers" to "Architects"

---

### Step 12: Watch the Build
```bash
oc logs -f bc/php-app
```

Wait for build to complete.

---

### Step 13: Wait for Deployment
```bash
oc get pods
```

Wait until `php-app-xxxxx` shows `Running`.

---

### Step 14: Verify Route
```bash
oc get route
```

**Expected output:**
```
NAME      HOST/PORT                                  PATH   SERVICES   PORT   TERMINATION   WILDCARD
php-app   php-app-indy.apps.ocp4.example.com               php-app    8080                 None
```

---

### Step 15: Test the Application
```bash
curl http://php-app-indy.apps.ocp4.example.com
```

**Expected output (in HTML):**
```html
<!DOCTYPE html>
<html>
<head>
    <title>Greeting Application</title>
    ...
</head>
<body>
    <h1>Namaste Architects!</h1>
    <p>This greeting is brought to you by OpenShift Templates</p>
</body>
</html>
```

---

### Step 16: Verify in Browser

Open: `http://php-app-indy.apps.ocp4.example.com`

**Should display:**
```
Namaste Architects!
```

---

## Success Criteria

- Project `indy` exists
- Template `ex288-php-app-template` has label `template: php-app`
- All parameters in template have `required: true`
- Application `php-app` is running
- Route `php-app-indy.apps.ocp4.example.com` is accessible
- Application displays: "Namaste Architects!" (not "Bonjour Engineers!")
- Git repository URL is correctly set in template

---

## Key Commands Reference
```bash
# List templates
oc get template
oc get template --show-labels

# Describe template (show parameters)
oc describe template <name>

# Process template (preview what would be created)
oc process <template-name>

# Process with parameters
oc process <template-name> -p PARAM1=value1 -p PARAM2=value2

# Deploy from template
oc new-app <template-name> -p PARAM1=value1 -p PARAM2=value2

# Create template from file
oc create -f <template.yaml>
oc apply -f <template.yaml>

# Delete template
oc delete template <name>

# Export existing resources as template
oc export dc,svc,route --as-template=my-template > template.yaml
```

---

## Understanding OpenShift Templates

### What are Templates?

Templates are reusable configurations that define a set of OpenShift resources (Deployments, Services, Routes, etc.) with parameterized values.

### Template Structure:
```yaml
apiVersion: template.openshift.io/v1
kind: Template
labels:                    # Labels for the template itself
  template: php-app
metadata:
  name: my-template
  annotations:            # Metadata shown in catalog
    description: "..."
parameters:               # Variables users can customize
  - name: NAME
    required: true
    value: default-value
objects:                  # Resources to create
  - apiVersion: v1
    kind: Service
    ...
  - apiVersion: apps/v1
    kind: Deployment
    ...
```
---