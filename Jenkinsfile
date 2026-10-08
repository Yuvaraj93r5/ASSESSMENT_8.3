pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'DOCKERHUB_USERNAME/placement-portal' // Replace with your Docker Hub username
        REGISTRY_CREDENTIALS = 'docker-hub-credentials'
        KUBE_CREDENTIALS = 'kubeconfig-credentials'
    }

    stages {
        stage('Clone Code') {
            steps {
                // Clones code automatically from the repository where this Jenkinsfile resides
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    echo "Building Docker Image..."
                    sh "docker build -t ${DOCKER_IMAGE}:${BUILD_NUMBER} ."
                    sh "docker tag ${DOCKER_IMAGE}:${BUILD_NUMBER} ${DOCKER_IMAGE}:latest"
                }
            }
        }

        stage('Test Application') {
            steps {
                script {
                    echo "Verifying image structure and containerized stability..."
                    // Start a short-lived test container to check if Nginx boots successfully
                    sh "docker run -d --name test_app -p 8080:80 ${DOCKER_IMAGE}:${BUILD_NUMBER}"
                    sh "sleep 5"
                    sh "curl -sI http://localhost:8080 | grep 'HTTP/1.1 200 OK'"
                    sh "docker rm -f test_app"
                }
            }
        }

        stage('Push Image') {
            steps {
                script {
                    // Authenticates and pushes to Docker Hub registry
                    withCredentials([usernamePassword(credentialsId: "${REGISTRY_CREDENTIALS}", usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh "echo ${DOCKER_PASS} | docker login -u ${DOCKER_USER} --password-stdin"
                        sh "docker push ${DOCKER_IMAGE}:${BUILD_NUMBER}"
                        sh "docker push ${DOCKER_IMAGE}:latest"
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                // Securely fetches kubeconfig file and executes rollout modifications
                configFileProvider([]) {
                    withCredentials([file(credentialsId: "${KUBE_CREDENTIALS}", variable: 'KUBECONFIG')]) {
                        sh "sed -i 's|DOCKERHUB_USERNAME/placement-portal:latest|${DOCKER_IMAGE}:${BUILD_NUMBER}|g' k8s/deployment.yaml"
                        sh "kubectl apply -f k8s/deployment.yaml --kubeconfig=${KUBECONFIG}"
                        sh "kubectl apply -f k8s/service.yaml --kubeconfig=${KUBECONFIG}"
                        sh "kubectl rollout status deployment/placement-portal-deployment --kubeconfig=${KUBECONFIG}"
                    }
                }
            }
        }
    }
    
    post {
        always {
            sh "docker logout"
        }
    }
}
