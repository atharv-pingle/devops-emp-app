pipeline {
    agent any

    environment {
        DOCKER_CREDENTIALS = credentials('docker-hub-credentials')
        BACKEND_IMAGE_NAME = "asp0217/employee-backend"
        FRONTEND_IMAGE_NAME = "asp0217/employee-frontend"
        GOPATH = "${WORKSPACE}/.go"
        NODE_MODULES_CACHE = "${WORKSPACE}/.node_modules"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/atharv-pingle/devops-emp-app.git'
            }
        }

        stage('Prepare Workspace') {
            steps {
                sh '''
                    mkdir -p ${GOPATH}/pkg
                    mkdir -p ${NODE_MODULES_CACHE}
                    chmod -R 777 ${GOPATH}
                    chmod -R 777 ${NODE_MODULES_CACHE}
                '''
            }
        }

        stage('Build Backend') {
            agent {
                docker {
                    image 'golang:1.20'
                    args "-v ${WORKSPACE}/backend:/go/src/app -w /go/src/app"
                    reuseNode true
                }
            }
            steps {
                dir('backend') {
                    sh '''
                        export GOPATH=/go
                        go mod download
                        go build -o app
                    '''
                }
            }
        }

        stage('Build Frontend') {
            agent {
                docker {
                    image 'node:18'
                    args "-v ${WORKSPACE}/frontend:/app -w /app"
                    reuseNode true
                }
            }
            steps {
                dir('frontend') {
                    sh '''
                        npm install
                        npm run build
                    '''
                }
            }
        }

        stage('Build and Push Docker Images') {
            steps {
                script {
                    // Build backend image
                    dir('backend') {
                        sh "docker build -t ${BACKEND_IMAGE_NAME}:${BUILD_NUMBER} ."
                    }

                    // Build frontend image
                    dir('frontend') {
                        sh "docker build -t ${FRONTEND_IMAGE_NAME}:${BUILD_NUMBER} ."
                    }

                    // Login to Docker Hub and push images
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh """
                            echo "${DOCKER_PASS}" | docker login -u "${DOCKER_USER}" --password-stdin
                            docker push ${BACKEND_IMAGE_NAME}:${BUILD_NUMBER}
                            docker push ${FRONTEND_IMAGE_NAME}:${BUILD_NUMBER}
                        """
                    }
                }
            }
        }

        stage('Update Deployment File') {
            environment {
                GIT_REPO_NAME = "devops-emp-app"
                GIT_USER_NAME = "atharv-pingle"
            }
            steps {
                withCredentials([string(credentialsId: 'github', variable: 'GITHUB_TOKEN')]) {
                    sh '''
                        git config user.email "atharvpingle@gmail.com"
                        git config user.name "atharv-pingle"
                        sed -i "s|replaceBackendImageTag|${BACKEND_IMAGE_NAME}:${BUILD_NUMBER}|g" backend/deployment.yml
                        sed -i "s|replaceFrontendImageTag|${FRONTEND_IMAGE_NAME}:${BUILD_NUMBER}|g" frontend/deployment.yml
                        git add backend/deployment.yml frontend/deployment.yml
                        git commit -m "Update backend and frontend images to latest version"
                        git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME} HEAD:main
                    '''
                }
            }
        }
    }

    post {
        always {
            // Cleanup
            sh '''
                docker rmi ${BACKEND_IMAGE_NAME}:${BUILD_NUMBER} || true
                docker rmi ${FRONTEND_IMAGE_NAME}:${BUILD_NUMBER} || true
                rm -rf ${WORKSPACE}/.npm-cache
            '''
            cleanWs()
        }
    }
}
