pipeline {
  environment {
    DOCKER_CREDENTIALS = credentials('docker-hub-credentials')
    BACKEND_IMAGE_NAME = "asp0217/employee-backend"
    FRONTEND_IMAGE_NAME = "asp0217/employee-frontend"
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
          image 'golang:1.20'  // Use an official Go Docker image with Go pre-installed
          args '-v $WORKSPACE/backend:/app -w /app'  // Mount backend directory into the container
        }
      }
      steps {
        sh 'ls -ltr' // List files to verify context
        sh 'go mod download' // Download Go dependencies
        sh 'go build -o app .' // Build the Go backend
      }
    }

    stage('Build Frontend') {
      agent {
        docker {
          image 'node:18'  // Use an official Node.js Docker image
          args '-v $WORKSPACE/frontend:/app -w /app'  // Mount frontend directory into the container
        }
      }
      steps {
        sh 'ls -ltr' // List files to verify context
        sh 'npm install' // Install Node dependencies
        sh 'npm run build' // Build the React frontend
      }
    }

    stage('Build Backend Docker Image') {
      steps {
        script {
          docker.build("${BACKEND_IMAGE_NAME}:${env.BUILD_NUMBER}", "-f backend/Dockerfile backend/")
        }
      }
    }

    stage('Build Frontend Docker Image') {
      steps {
        script {
          docker.build("${FRONTEND_IMAGE_NAME}:${env.BUILD_NUMBER}", "-f frontend/Dockerfile frontend/")
        }
      }
    }

    stage('Push Docker Images') {
      steps {
        script {
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
            sed -i "s/replaceBackendImageTag/${BACKEND_IMAGE_NAME}:${env.BUILD_NUMBER}/g" backend/deployment.yml
            sed -i "s/replaceFrontendImageTag/${FRONTEND_IMAGE_NAME}:${env.BUILD_NUMBER}/g" frontend/deployment.yml
            git add backend/deployment.yml frontend/deployment.yml
            git commit -m "Update backend and frontend images to latest version"
            git push https://${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME} HEAD:main
          '''
        }
      }
    }
  }
}
