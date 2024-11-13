pipeline {
    agent {
        docker {
            image 'docker:dind'  // Docker-in-Docker image
            args '--privileged -v /var/run/docker.sock:/var/run/docker.sock'  // Allow Docker commands
        }
    }

    environment {
        DOCKER_CREDENTIALS = credentials('docker-hub-credentials')
        BACKEND_IMAGE_NAME = "asp0217/employee-backend"
        FRONTEND_IMAGE_NAME = "asp0217/employee-frontend"
        // Added cache directories for dependencies
        GOPATH = "${WORKSPACE}/.go"
        NODE_MODULES_CACHE = "${WORKSPACE}/.node_modules"
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
                    args """
                        -v ${env.WORKSPACE}/backend:/app 
                        -v ${env.GOPATH}:/go 
                        -w /app
                        --network host
                    """
                    reuseNode true
                }
            }
            steps {
                sh '''
                    # Create cache directory if it doesn't exist
                    mkdir -p "${GOPATH}/pkg"
                    
                    # Download dependencies with caching
                    GOPATH=${GOPATH} go mod download
                    
                    # Build the application
                    GOPATH=${GOPATH} go build -o app .
                '''
            }
        }

        stage('Build Frontend') {
            agent {
                docker {
                    image 'node:18'
                    args """
                        -v ${env.WORKSPACE}/frontend:/app 
                        -v ${env.NODE_MODULES_CACHE}:/app/node_modules 
                        -w /app
                        --network host
                    """
                    reuseNode true
                }
            }
            steps {
                sh '''
                    # Use cached node_modules if available
                    if [ -d "${NODE_MODULES_CACHE}" ]; then
                        echo "Using cached node_modules"
                    else
                        echo "Installing dependencies"
                        npm install
                        # Cache the node_modules
                        cp -r node_modules/* ${NODE_MODULES_CACHE}/
                    fi
                    
                    # Build the application
                    npm run build
                '''
            }
        }

        stage('Build and Push Docker Images') {
            steps {
                script {
                    // Build backend image with cache
                    docker.build("${BACKEND_IMAGE_NAME}:${env.BUILD_NUMBER}", 
                        "--build-arg GOPATH=${GOPATH} -f backend/Dockerfile backend/")

                    // Build frontend image with cache
                    docker.build("${FRONTEND_IMAGE_NAME}:${env.BUILD_NUMBER}", 
                        "--build-arg NODE_MODULES_CACHE=${NODE_MODULES_CACHE} -f frontend/Dockerfile frontend/")

                    // Push images
                    docker.withRegistry('', 'docker-hub-credentials') {
                        docker.image("${BACKEND_IMAGE_NAME}:${env.BUILD_NUMBER}").push()
                        docker.image("${FRONTEND_IMAGE_NAME}:${env.BUILD_NUMBER}").push()
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
                        sed -i "s|replaceBackendImageTag|${BACKEND_IMAGE_NAME}:${env.BUILD_NUMBER}|g" backend/deployment.yml
                        sed -i "s|replaceFrontendImageTag|${FRONTEND_IMAGE_NAME}:${env.BUILD_NUMBER}|g" frontend/deployment.yml
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
            // Cleanup cached dependencies if needed
            sh '''
                rm -rf ${GOPATH}
                rm -rf ${NODE_MODULES_CACHE}
            '''
        }
    }
}
