markdown
# Sample DevOps App 🚀

A minimal Java web application deployed on a local **k3s Kubernetes cluster** using **Helm** and automated via a **Jenkins pipeline**.  
This project demonstrates end-to-end CI/CD with Docker, Helm, and Jenkins.

---

## 📦 Project Overview

- **App Name:** `sample-devops-app`
- **Stack:** Java + Docker + k3s + Helm + Jenkins
- **Deployment:** Minimal Helm chart (Deployment + Service only)
- **Pipeline:** Jenkinsfile automates build, import, deploy, and smoke test

---

## 🛠️ Prerequisites

- Ubuntu VM (tested on 20.04+)
- Docker installed and running
- k3s installed (`curl -sfL https://get.k3s.io | sh -`)
- Helm installed (`curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash`)
- Jenkins installed and configured
- GitHub repository cloned locally

---

## 🔹 Step 1: Clone the Repository

```bash
git clone https://github.com/mylab12345/sample-app.git
cd sample-app
🔹 Step 2: Build Docker Image
bash
docker build -t sample-devops-app:0.1.0 .
docker image ls | grep sample-devops-app
🔹 Step 3: Import Image into k3s
bash
docker save sample-devops-app:0.1.0 -o sample-devops-app_0.1.0.tar
sudo k3s ctr images import sample-devops-app_0.1.0.tar
sudo k3s crictl images | grep sample-devops-app
🔹 Step 4: Create Minimal Helm Chart
bash
mkdir helm-app
cd helm-app
Chart.yaml

yaml
apiVersion: v2
name: helm-app
description: A minimal Helm chart for sample-app
type: application
version: 0.1.0
appVersion: "0.1.0"
values.yaml

yaml
replicaCount: 1

image:
  repository: sample-devops-app
  tag: "0.1.0"
  pullPolicy: IfNotPresent

service:
  type: NodePort
  port: 80
  nodePort: 30080
templates/deployment.yaml

yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
spec:
  replicas: {{ .Values.replicaCount }}
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
          image: {{ .Values.image.repository }}:{{ .Values.image.tag }}
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: 80
templates/service.yaml

yaml
apiVersion: v1
kind: Service
metadata:
  name: sample-app
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: 80
      nodePort: {{ .Values.service.nodePort }}
  selector:
    app: sample-app
🔹 Step 5: Deploy with Helm
bash
kubectl create ns web
helm upgrade --install sample-app ./helm-app -n web
kubectl -n web get pods
kubectl -n web get svc
Test the app:

bash
curl http://127.0.0.1:30080
🔹 Step 6: Jenkins Pipeline
Jenkinsfile

groovy
pipeline {
    agent any
    environment {
        APP_NAME   = 'sample-devops-app'
        APP_TAG    = "0.1.${BUILD_NUMBER}"
        NAMESPACE  = 'web'
        KUBECONFIG = '/etc/rancher/k3s/k3s.yaml'
    }
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/mylab12345/sample-app.git'
            }
        }
        stage('Build Image') {
            steps {
                sh '''
                  docker build -t ${APP_NAME}:${APP_TAG} .
                  docker image ls | grep ${APP_NAME}
                '''
            }
        }
        stage('Import to k3s') {
            steps {
                sh '''
                  docker save ${APP_NAME}:${APP_TAG} -o ${APP_NAME}_${APP_TAG}.tar
                  sudo k3s ctr images import ${APP_NAME}_${APP_TAG}.tar
                  sudo k3s crictl images | grep ${APP_NAME}
                '''
            }
        }
        stage('Helm Upgrade') {
            steps {
                sh '''
                  sed -i "s/tag: \\".*\\"/tag: \\"${APP_TAG}\\"/g" helm-app/values.yaml
                  kubectl get ns ${NAMESPACE} || kubectl create ns ${NAMESPACE}
                  helm uninstall sample-app -n ${NAMESPACE} || true
                  rm -f helm-app-*.tgz
                  helm upgrade --install sample-app ./helm-app -n ${NAMESPACE}
                  kubectl -n ${NAMESPACE} rollout status deploy/sample-app --timeout=60s
                '''
            }
        }
        stage('Smoke Test') {
            steps {
                sh 'curl -sSf http://127.0.0.1:30080 | head -n 5'
            }
        }
    }
    post {
        success { echo '✅ Deployment successful!' }
        failure { echo '❌ Build or deploy failed.' }
    }
}
✅ Final Result
Jenkins builds Docker image with unique tag (0.1.$BUILD_NUMBER).

Image imported into k3s container runtime.

Helm chart deploys app into namespace web.

Smoke test verifies app is running on NodePort 30080.
