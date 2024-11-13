pipeline {
    agent any

    environment {
        DOCKER_CREDENTIALS = credentials('docker-hub-credentials')
        BACKEND_IMAGE = "asp0217/employee-backend:${BUILD_NUMBER}"
        FRONTEND_IMAGE = "asp0217/employee-frontend:${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/atharv-pingle/devops-emp-app.git'
            }
        }

        stage('Build Backend') {
            agent {
                docker {
                    image 'golang:1.20'
                    reuseNode true
                }
            }
            steps {
                dir('backend') {
                    sh """
                        export GOCACHE=/tmp/.cache/go-build
                        export GOPATH=/tmp/go
                        go mod download
                        CGO_ENABLED=0 GOOS=linux go build -o app
                    """
                }
            }
        }

        stage('Build Frontend') {
            agent {
                docker {
                    image 'node:18'
                    reuseNode true
                }
            }
            steps {
                dir('frontend') {
                    sh 'npm ci && npm run build'
                }
            }
        }

        stage('Build & Push Docker Images') {
            steps {
                script {
                    // Build Images
                    sh """
                        cd backend && docker build -t ${BACKEND_IMAGE} .
                        cd ../frontend && docker build -t ${FRONTEND_IMAGE} .
                    """

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

        stage('Update Deployment Files') {
            steps {
                withCredentials([string(credentialsId: 'github', variable: 'GITHUB_TOKEN')]) {
                    sh """
                        git config user.email "atharvpingle@gmail.com"
                        git config user.name "atharv-pingle"
                        
                        sed -i "s|replaceBackendImageTag|${BACKEND_IMAGE}|g" backend/deployment.yml
                        sed -i "s|replaceFrontendImageTag|${FRONTEND_IMAGE}|g" frontend/deployment.yml
                        
                        git add backend/deployment.yml frontend/deployment.yml
                        git commit -m "Update deployment images to version ${BUILD_NUMBER}"
                        git push https://\${GITHUB_TOKEN}@github.com/atharv-pingle/devops-emp-app HEAD:main
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
            """
            cleanWs()
        }
    }
}
