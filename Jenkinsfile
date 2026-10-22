pipeline {
    agent any
    environment {
        DOCKER_IMAGE = 'subitcha1998/trend-app'
        AWS_REGION   = 'us-east-1'
        CLUSTER_NAME = 'trend-cluster'
    }
    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/srisubi1998-ctrl/Trend.git'
            }
        }
        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE}:latest ."
            }
        }
        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USER')]) {
                    sh "echo ${DOCKER_PASSWORD} | docker login -u ${DOCKER_USER} --password-stdin"
                    sh "docker push ${DOCKER_IMAGE}:latest"
                }
            }
        }
        stage('Deploy to EKS') {
            steps {
                sh """
                    aws eks update-kubeconfig --region ${AWS_REGION} --name ${CLUSTER_NAME}
                    kubectl apply -f k8s/
                    kubectl rollout restart deployment/trend-app
                    kubectl rollout status deployment/trend-app --timeout=180s
                    kubectl get pods
                    kubectl get svc trend-service
                """
            }
        }
    }
    post {
        always {
            sh 'docker image prune -f || true'
        }
        success { echo 'Deployed to EKS successfully' }
        failure { echo 'Pipeline failed' }
    }
}
