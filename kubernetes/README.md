# Frappe CRM Deployment on Kubernetes

## 1. Project Overview

I already had Frappe CRM running locally using Docker Compose. As part of this task, I deployed the same application on a local Kubernetes cluster using Minikube.

I kept the existing application code and Docker Compose setup unchanged.

**Application URLs**

- Docker Compose: http://crm.localhost:8000
- Kubernetes: http://crm.localhost:8080

## 2. Deployment Steps

### Step 1: Start Minikube

Command:

    minikube start --driver=docker --cpus=4 --memory=4096 --disk-size=30g

**Why:** To create a local Kubernetes cluster using Docker.

**Result:** Minikube started and provided a Kubernetes node.

### Step 2: Create a namespace

Command:

    kubectl apply -f kubernetes/namespace.yaml

**Why:** To keep all CRM resources separate from other Kubernetes resources.

**Result:** Created the `frappe-crm` namespace.

### Step 3: Configure application settings and passwords

I created a ConfigMap for non-sensitive settings such as database and Redis hostnames. I also created Kubernetes Secrets for the database root password and CRM administrator password.

**Why:** To separate configuration from application manifests and avoid storing actual passwords in YAML files.

**Result:** Kubernetes resources can access the required configuration and passwords.

### Step 4: Deploy MariaDB and Redis

I created the MariaDB and Redis Deployments and Services.

**Why:** Frappe CRM requires MariaDB for database storage and Redis for caching and queue services.

**Result:** Both services became available inside the Kubernetes namespace.

### Step 5: Configure persistent storage

I created PersistentVolumeClaims for MariaDB data and Frappe site files.

**Why:** To retain data even if application containers restart.

**Result:** Both 5Gi storage claims became Bound.

### Step 6: Initialize the Frappe site

I used the `frappe-site-init` Kubernetes Job to initialize the site and install the CRM application.

**Why:** Frappe needs an initialized site before it can serve application requests.

**Result:** The initialization Job completed successfully.

### Step 7: Deploy Frappe CRM and Nginx

I deployed Frappe CRM with an Nginx sidecar container.

**Why:** Frappe handles application requests, while Nginx serves static CSS and JavaScript files and forwards other requests to Frappe.

**Result:** Both containers became Ready.

### Step 8: Access the application

Command:

    kubectl -n frappe-crm port-forward service/frappe-crm 8080:8000

**Why:** To forward local port 8080 to the Kubernetes Service.

**Result:** The application became accessible at:

http://crm.localhost:8080

Keep the port-forward command running while using the application.

## 3. Errors Faced and Fixes

### Error 1: localhost does not exist

**Problem:** Opening `http://localhost:8080` showed a Frappe site not found message.

**Reason:** The Frappe site was configured with the hostname `crm.localhost`.

**Fix:** I opened the application using `http://crm.localhost:8080`.

### Error 2: CSS and JavaScript files returned 404

**Problem:** The application loaded without its proper styling and static assets.

**Reason:** Frappe's asset paths used links that were not available inside the separate Nginx container.

**Fix:** I added an init container to copy Frappe and CRM public assets into a shared `emptyDir` volume. Nginx serves the copied files.

**Verification:** CSS and JavaScript requests returned HTTP 200.

## 4. Kubernetes YAML Files

| File | Why I created it / Its use |
|---|---|
| `namespace.yaml` | Creates a separate namespace for CRM resources. |
| `configmap.yaml` | Stores non-sensitive database and Redis settings. |
| `secret.example.yaml.sample` | Provides an example Secret structure without real passwords. |
| `mariadb.yaml` | Creates the MariaDB Deployment, Service and database storage claim. |
| `redis.yaml` | Creates the Redis Deployment and Service. |
| `sites-pvc.yaml` | Provides persistent storage for Frappe site files. |
| `frappe-init-job.yaml` | Initializes the Frappe site and installs the CRM application. |
| `deployment.yaml` | Runs Frappe CRM and Nginx, including static asset preparation. |
| `nginx-configmap.yaml` | Configures Nginx to serve static assets and forward application requests. |
| `service.yaml` | Exposes the CRM inside Kubernetes and routes traffic to Nginx. |

**Note:** The Secret example is only a template. Actual Kubernetes Secrets were created separately. Passwords are not included in this repository.

## 5. Verification Commands

### Validate Kubernetes YAML files

Command:

    kubectl apply --dry-run=client -f kubernetes/

**Why:** To check the manifests without changing the running deployment.

**Result:** Kubernetes recognised the manifests without YAML parsing errors.

### Check Kubernetes resources

Command:

    kubectl get pods,deployments,services,pvc,jobs -n frappe-crm

**Why:** To check the status of Pods, Deployments, Services, storage and initialization Job.

**Result:**
- Frappe CRM and Nginx: 2/2 containers Running.
- MariaDB: Running.
- Redis: Running.
- Frappe initialization Job: Complete.
- Both 5Gi PVCs: Bound.

## 6. Final Result

Frappe CRM is accessible through Minikube at `http://crm.localhost:8080`.

The original Docker Compose deployment continues to work at `http://crm.localhost:8000`.

This is a local Minikube deployment for learning and testing. Separate background worker, scheduler and WebSocket services have not been deployed as part of this setup.
