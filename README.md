# Deploying to Kubernetes with Argo CD and k3d

## Introduction

This document demonstrates the implementation of a GitOps workflow using Argo CD for continuous deployment to a Kubernetes cluster. I've followed the approach outlined in the article "Application Deploy to Kubernetes with Argo CD and k3d" to set up a local Kubernetes environment with k3d and deploy a sample application using Argo CD.

## 1. Setting Up k3d Cluster

k3d is a lightweight wrapper to run k3s (a minimized Kubernetes distribution) in Docker. I created a Kubernetes cluster using the following command:

```bash
k3d cluster create argocd-cluster --api-port 127.0.0.1:6550 -p "8081:80@loadbalancer" -p "8444:443@loadbalancer"
```

After creating the cluster, I verified it was running correctly:

```bash
kubectl get nodes
```

<img width="868" alt="Screenshot 2025-04-09 at 2 04 47 PM" src="https://github.com/user-attachments/assets/cdf69bdb-f851-4ef8-bf0c-6664ff274191" />


## 2. Installing Argo CD

I created a dedicated namespace for Argo CD:

```bash
kubectl create namespace argocd
```

Then I applied the Argo CD installation manifest:

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

After a few minutes, I verified that all Argo CD pods were running correctly:

```bash
kubectl get pods -n argocd
```

<img width="893" alt="Screenshot 2025-04-09 at 2 05 22 PM" src="https://github.com/user-attachments/assets/c177dfe2-bc1b-4220-886e-45569632e65b" />

<img width="804" alt="Screenshot 2025-04-09 at 2 05 46 PM" src="https://github.com/user-attachments/assets/5f9e6c9e-9d06-4e66-8d5a-9297caaadb30" />


## 3. Accessing Argo CD UI

To access the Argo CD UI, I set up port forwarding:

```bash
kubectl port-forward svc/argocd-server -n argocd 8090:443
```

I retrieved the initial admin password using:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

<img width="1685" alt="Screenshot 2025-04-09 at 2 06 07 PM" src="https://github.com/user-attachments/assets/428b35d4-7f72-42cc-800f-47940ba386b1" />

<img width="961" alt="Screenshot 2025-04-09 at 2 06 50 PM" src="https://github.com/user-attachments/assets/b4069a60-e6cc-4e30-9376-fc645c2f8a37" />


## 4. Creating a Sample Application

I created a simple web application with the following structure:

```
sample-app/
├── index.html
├── Dockerfile
└── k8s/
    ├── deployment.yaml
    ├── service.yaml
    └── ingress.yaml
```

### index.html
```html
<!DOCTYPE html>
<html>
<head>
    <title>Sample App for Argo CD</title>
</head>
<body>
    <h1>Hello from Argo CD!</h1>
    <p>This is a sample application deployed using Argo CD.</p>
</body>
</html>
```

### Dockerfile
```Dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
```

### k8s/deployment.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
  labels:
    app: sample-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
      - name: sample-app
        image: nginx:alpine
        ports:
        - containerPort: 80
```

### k8s/service.yaml
```yaml
apiVersion: v1
kind: Service
metadata:
  name: sample-app
spec:
  selector:
    app: sample-app
  ports:
  - port: 80
    targetPort: 80
  type: ClusterIP
```

### k8s/ingress.yaml
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sample-app
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: sample-app
            port:
              number: 80
```

## 5. Setting Up Git Repository

I initialized a Git repository for the application code:

```bash
git init
git add .
git commit -m "Initial commit"
```

I created a GitHub repository and pushed my code:

```bash
git remote add origin https://github.com/zmehta2/sample-app.git
git branch -M main
git push -u origin main
```

https://github.com/zmehta2/sample-app/ 

## 6. Configuring Argo CD

I logged into the Argo CD UI and created a new application with the following details:

- Application Name: sample-app
- Project: default
- Sync Policy: Automatic
- Repository URL: https://github.com/zmehta2/sample-app.git
- Path: k8s
- Cluster URL: https://kubernetes.default.svc
- Namespace: default
  
<img width="1104" alt="Screenshot 2025-04-09 at 2 11 13 PM" src="https://github.com/user-attachments/assets/1169c7c8-298a-41c4-ba26-3d35f1cf0369" />

After creating the application, Argo CD synchronized the manifests from the Git repository and deployed the application to the Kubernetes cluster.

<img width="474" alt="Screenshot 2025-04-09 at 2 11 43 PM" src="https://github.com/user-attachments/assets/31a57f6d-1af1-4614-a28f-6598aef8bbeb" />

<img width="1468" alt="Screenshot 2025-04-09 at 2 12 19 PM" src="https://github.com/user-attachments/assets/c9550174-5b69-4b73-b131-64e4f9186cef" />

## 7. Verifying the Deployment

I verified that the application was successfully deployed:

```bash
kubectl get pods,svc,ingress
```

<img width="765" alt="Screenshot 2025-04-09 at 2 13 15 PM" src="https://github.com/user-attachments/assets/2b0acf37-6c16-4543-b9f8-e2af13ab79af" />

To access the application, I set up port forwarding:

```bash
kubectl port-forward svc/sample-app 8082:80
```

<img width="630" alt="Screenshot 2025-04-09 at 2 13 38 PM" src="https://github.com/user-attachments/assets/4867d98c-a84a-40e2-a2d8-c8ab6d9f8b98" />

## 8. Demonstrating GitOps Workflow

To demonstrate the GitOps workflow, I modified the `index.html` file to update the content:

```html
<!DOCTYPE html>
<html>
<head>
    <title>Updated Sample App for Argo CD</title>
</head>
<body>
    <h1>Hello from Argo CD!</h1>
    <p>This is an updated sample application deployed using Argo CD.</p>
    <p>GitOps workflow automatically deployed this change!</p>
</body>
</html>
```

I committed and pushed the changes:

```bash
git add index.html
git commit -m "Update index.html content"
git push origin main
```

Argo CD automatically detected the changes in the Git repository and updated the deployment.

<img width="1475" alt="Screenshot 2025-04-09 at 2 19 00 PM" src="https://github.com/user-attachments/assets/af5594ac-ce01-49b9-a0e0-060d68a7db15" />

After the sync completed, I refreshed the application in the browser to see the updated content.

<img width="753" alt="Screenshot 2025-04-09 at 2 21 32 PM" src="https://github.com/user-attachments/assets/6a191ecd-5d54-43c9-8715-850a65e25018" />

