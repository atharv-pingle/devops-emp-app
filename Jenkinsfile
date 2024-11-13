pipeline {
    agent {
        docker {
            image 'docker:dind'
            args '--privileged -v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
    
    environment {
        DOCKER_HUB_CREDENTIALS = credentials('docker-hub-credentials')
        DOCKER_IMAGE_FRONTEND = "asp0217/employee-frontend"
        DOCKER_IMAGE_BACKEND = "asp0217/employee-backend"
        GIT_REPO_URL = "https://github.com/atharv-pingle/devops-emp-app.git"
    }
    
    stages {
        stage('Install Tools') {
            steps {
                sh '''
                    # Install basic utilities including curl first
                    apk add --no-cache curl git sudo bash socat

                    # Get EC2 Public IP after curl is installed
                    export EC2_PUBLIC_IP=$(curl -s http://169.254.169.254/latest/meta-data/public-ipv4)

                    # Install kubectl
                    curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
                    chmod +x kubectl
                    mv kubectl /usr/local/bin/

                    # Install Minikube
                    curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
                    chmod +x minikube
                    mv minikube /usr/local/bin/

                    # Install ArgoCD CLI
                    curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
                    chmod +x argocd
                    mv argocd /usr/local/bin/
                '''
            }
        }

        stage('Checkout') {
            steps {
                cleanWs()
                git branch: 'main', url: env.GIT_REPO_URL
            }
        }
        
        stage('Start Minikube') {
            steps {
                sh '''
                    # Start Minikube with Docker driver
                    minikube start --driver=docker --force
                    
                    # Verify Minikube status
                    minikube status
                    
                    # Enable ingress addon
                    minikube addons enable ingress
                    
                    # Point shell to minikube's Docker daemon
                    eval $(minikube -p minikube docker-env)
                '''
            }
        }
        
        stage('Build and Push Docker Images') {
            steps {
                script {
                    // Docker Hub Login
                    sh "echo ${DOCKER_HUB_CREDENTIALS_PSW} | docker login -u ${DOCKER_HUB_CREDENTIALS_USR} --password-stdin"
                    
                    // Build Frontend
                    dir('frontend') {
                        sh """
                            docker build -t ${DOCKER_IMAGE_FRONTEND}:${env.BUILD_NUMBER} .
                            docker tag ${DOCKER_IMAGE_FRONTEND}:${env.BUILD_NUMBER} ${DOCKER_IMAGE_FRONTEND}:latest
                            docker push ${DOCKER_IMAGE_FRONTEND}:${env.BUILD_NUMBER}
                            docker push ${DOCKER_IMAGE_FRONTEND}:latest
                        """
                    }
                    
                    // Build Backend
                    dir('backend') {
                        sh """
                            docker build -t ${DOCKER_IMAGE_BACKEND}:${env.BUILD_NUMBER} .
                            docker tag ${DOCKER_IMAGE_BACKEND}:${env.BUILD_NUMBER} ${DOCKER_IMAGE_BACKEND}:latest
                            docker push ${DOCKER_IMAGE_BACKEND}:${env.BUILD_NUMBER}
                            docker push ${DOCKER_IMAGE_BACKEND}:latest
                        """
                    }
                }
            }
        }
        
        stage('Deploy ArgoCD') {
            steps {
                sh '''
                    # Create ArgoCD namespace
                    kubectl create namespace argocd --dry-run=client -o yaml | kubectl apply -f -
                    
                    # Install ArgoCD
                    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
                    
                    # Configure ArgoCD server as NodePort
                    kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort", "ports": [{"port": 443, "targetPort": 8080, "nodePort": 30080}]}}'
                    
                    # Wait for ArgoCD server to be ready
                    kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd
                    
                    # Port forward in background for local access
                    nohup kubectl port-forward svc/argocd-server -n argocd 8080:443 --address='0.0.0.0' &
                    
                    # Wait for port-forward to be ready
                    sleep 10
                '''
            }
        }
        
        stage('Configure ArgoCD') {
            steps {
                sh '''
                    # Get EC2 Public IP for the configuration
                    EC2_PUBLIC_IP=$(curl -s http://169.254.169.254/latest/meta-data/public-ipv4)
                    
                    # Wait for password secret
                    kubectl wait --for=condition=available --timeout=300s secret/argocd-initial-admin-secret -n argocd
                    
                    # Get ArgoCD password
                    ARGOCD_PASSWORD=$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d)
                    
                    # Login to ArgoCD using localhost (since we port-forwarded)
                    argocd login localhost:8080 --username admin --password $ARGOCD_PASSWORD --insecure
                    
                    # Create ArgoCD application
                    argocd app create devops-emp-app \
                        --repo ${GIT_REPO_URL} \
                        --path k8s \
                        --dest-server https://kubernetes.default.svc \
                        --dest-namespace default \
                        --sync-policy automated
                    
                    # Print access information
                    echo "================================================================"
                    echo "Access Information:"
                    echo "ArgoCD UI: https://${EC2_PUBLIC_IP}:30080"
                    echo "Default username: admin"
                    echo "Initial password: ${ARGOCD_PASSWORD}"
                    echo "================================================================"
                    
                    # Create Ingress for the application
                    cat <<EOF | kubectl apply -f -
                    apiVersion: networking.k8s.io/v1
                    kind: Ingress
                    metadata:
                      name: app-ingress
                      annotations:
                        nginx.ingress.kubernetes.io/rewrite-target: /
                    spec:
                      rules:
                      - host: app.${EC2_PUBLIC_IP}.nip.io
                        http:
                          paths:
                          - path: /
                            pathType: Prefix
                            backend:
                              service:
                                name: frontend-service
                                port:
                                  number: 80
                    EOF
                    
                    echo "Application will be available at: http://app.${EC2_PUBLIC_IP}.nip.io"
                '''
            }
        }
    }
    
    post {
        always {
            node {  // Added node block for post actions
                sh '''
                    # Kill port-forward process
                    pkill kubectl || true
                    docker logout
                '''
                cleanWs()  // Moved cleanWs outside of sh block
            }
        }
    }
}
