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
                  helm upgrade --install sample-app helm-app -n ${NAMESPACE}
                  kubectl -n ${NAMESPACE} rollout status deploy/sample-app
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
