pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = "asp0217"
        DOCKER_CREDENTIALS = credentials('docker-hub-credentials')
        BACKEND_IMAGE = "${DOCKER_REGISTRY}/employee-backend:${BUILD_NUMBER}"
        FRONTEND_IMAGE = "${DOCKER_REGISTRY}/employee-frontend:${BUILD_NUMBER}"
        NPM_CONFIG_CACHE = "${WORKSPACE}/.npm"
        GITHUB_REPO = "atharv-pingle/devops-emp-app"
        // Set Go environment variables
        GOCACHE = "${WORKSPACE}/.gocache"
        GOPATH = "${WORKSPACE}/.go"
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', 
                    url: "https://github.com/${GITHUB_REPO}.git"
            }
        }
        
        stage('Build Backend') {
            agent {
                docker {
                    image 'golang:1.20'
                    reuseNode true
                    args """
                        -u root:root
                        -e GOCACHE=${WORKSPACE}/.gocache
                        -e GOPATH=${WORKSPACE}/.go
                        -e GO111MODULE=on
                        -v ${WORKSPACE}/.gocache:${WORKSPACE}/.gocache
                        -v ${WORKSPACE}/.go:${WORKSPACE}/.go
                    """
                }
            }
            steps {
                dir('backend') {
                    sh '''
                        # Create cache directories
                        mkdir -p ${GOCACHE}
                        mkdir -p ${GOPATH}
                        
                        # Download dependencies and build
                        go mod download
                        go mod verify
                        CGO_ENABLED=0 GOOS=linux go build -buildvcs=false -o app
                    '''
                }
            }
        }
        
        stage('Build Frontend') {
            agent {
                docker {
                    image 'node:18-alpine'
                    reuseNode true
                    args """
                        -u root:root
                        -e HOME=${WORKSPACE}
                    """
                }
            }
            steps {
                dir('frontend') {
                    sh '''
                        mkdir -p ${NPM_CONFIG_CACHE}
                        npm config set cache ${NPM_CONFIG_CACHE}
                        npm ci
                        npm run build
                    '''
                }
            }
        }
        
        stage('Build & Push Docker Images') {
            steps {
                script {
                    // Build Backend Image
                    dir('backend') {
                        sh "docker build -t ${BACKEND_IMAGE} ."
                    }
                    
                    // Build Frontend Image
                    dir('frontend') {
                        sh "docker build -t ${FRONTEND_IMAGE} ."
                    }
                    
                    // Push Images
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh """
                            echo "${DOCKER_PASS}" | docker login -u "${DOCKER_USER}" --password-stdin
                            docker push ${BACKEND_IMAGE}
                            docker push ${FRONTEND_IMAGE}
                        """
                    }
                }
            }
        }
        
        stage('Update Kubernetes Manifests') {
            steps {
                withCredentials([string(credentialsId: 'github', variable: 'GITHUB_TOKEN')]) {
                    sh """
                        git config user.email "atharvpingle@gmail.com"
                        git config user.name "atharv-pingle"
                        
                        # Update image tags in k8s manifests
                        sed -i "s|image: ${DOCKER_REGISTRY}/employee-backend:.*|image: ${BACKEND_IMAGE}|g" k8s/k8s.yaml
                        sed -i "s|image: ${DOCKER_REGISTRY}/employee-frontend:.*|image: ${FRONTEND_IMAGE}|g" k8s/k8s.yaml
                        
                        # Commit and push changes
                        git add k8s/k8s.yml
                        git commit -m "Update k8s deployment images to version ${BUILD_NUMBER}" || true
                        git push https://\${GITHUB_TOKEN}@github.com/${GITHUB_REPO}.git HEAD:main
                    """
                }
            }
        }
    }
    
    post {
        always {
            sh """
                docker rmi ${BACKEND_IMAGE} || true
                docker rmi ${FRONTEND_IMAGE} || true
                docker system prune -f
                rm -rf ${WORKSPACE}/.gocache
                rm -rf ${WORKSPACE}/.go
                rm -rf ${WORKSPACE}/.npm
            """
            cleanWs()
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
