pipeline {
    agent any
    environment {
        DOCKER_CREDENTIALS = credentials('docker-hub-credentials')
        BACKEND_IMAGE = "asp0217/employee-backend:${BUILD_NUMBER}"
        FRONTEND_IMAGE = "asp0217/employee-frontend:${BUILD_NUMBER}"
        NPM_CONFIG_CACHE = "${WORKSPACE}/.npm"
       
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
                    args '-u root:root' // Run as root to avoid permission issues
                }
            }
            steps {
                dir('frontend') {
                    sh '''
                        mkdir -p ${NPM_CONFIG_CACHE}
                        chown -R $(id -u):$(id -g) ${NPM_CONFIG_CACHE}
                        npm config set cache ${NPM_CONFIG_CACHE}
                        npm ci --unsafe-perm
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
        stage('Update k8s Deployment File') {
            steps {
                withCredentials([string(credentialsId: 'github', variable: 'GITHUB_TOKEN')]) {
                    sh """
                        git config user.email "atharvpingle@gmail.com"
                        git config user.name "atharv-pingle"
                        
                        sed -i "s|asp0217/employee-backend|${BACKEND_IMAGE}|g" k8s/k8s.yml
                        sed -i "s|asp0217/employee-frontend|${BACKEND_IMAGE}|g" k8s/k8s.yml
                        
                        git add k8s/k8s.yml
                        git commit -m "Update k8s deployment images to version ${BUILD_NUMBER}"
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
