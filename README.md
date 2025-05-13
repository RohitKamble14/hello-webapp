
# 🚀 Deploying a Static index.html to EC2 using Kind + GitHub Actions

This guide documents how I deployed a simple static `index.html` (To-Do List app) to an EC2 instance using Docker, Kubernetes (Kind), and GitHub Actions for CI/CD.

---

## ✅ Overview

- **Frontend**: Static `index.html` (My To-Do List)
- **Containerization**: Docker
- **CI/CD**: GitHub Actions
- **Deployment**: Kind (Kubernetes IN Docker) on AWS EC2
- **Exposure**: NodePort Service on port 30080

---

## 📁 Project Structure

```
todo-app/
├── index.html
├── Dockerfile
├── .github/
│   └── workflows/
│       └── deploy.yml
└── k8s/
    ├── deployment.yml
    └── service.yml
```

---

## 🐳 Dockerfile

```Dockerfile
FROM nginx:alpine
RUN rm -rf /usr/share/nginx/html/*
COPY index.html /usr/share/nginx/html/
EXPOSE 80
```

---

## 🛠️ GitHub Actions Workflow (.github/workflows/deploy.yml)

```yaml
name: Deploy to Kind on EC2
on:
  push:
    branches: [ main ]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Login to DockerHub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build & Push image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/todo-app:latest

      - name: Deploy to EC2 via SSH
        uses: appleboy/ssh-action@v0.1.7
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ${{ secrets.EC2_USER }}
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            cd ~
            kubectl delete deployment todo-app --ignore-not-found
            kubectl apply -f todo-k8s/deployment.yml
            kubectl apply -f todo-k8s/service.yml
            kubectl rollout status deployment/todo-app
```

---

## ⚙️ Kubernetes Manifests

**deployment.yml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: todo-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: todo
  template:
    metadata:
      labels:
        app: todo
    spec:
      containers:
        - name: todo-app
          image: <your-dockerhub-username>/todo-app:latest
          ports:
            - containerPort: 80
```

**service.yml**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: todo-service
spec:
  type: NodePort
  selector:
    app: todo
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
      nodePort: 30080
```

---

## ☁️ EC2 Setup Summary

1. **Install Docker, Kind, kubectl**
2. **Create Kind Cluster with Port Mapping**

`kind-config.yaml`
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30080
        hostPort: 30080
        protocol: TCP
```

```bash
kind create cluster --config kind-config.yaml --name todo-cluster
```

3. **Create `~/todo-k8s/` and add `deployment.yml` + `service.yml`**
4. **Allow port 30080 in EC2 Security Group**

---

## 🌐 Access the App

Visit in browser:

```
http://<your-ec2-ip>:30080
```

---

## ✅ Done!
Every Git push to `main` triggers an auto-deploy to your EC2 instance using Kind. CI/CD + K8s + GitHub Actions — production-style!
