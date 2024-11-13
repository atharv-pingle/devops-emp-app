pipeline {
  agent any
  
  environment {
    DOCKER_CREDENTIALS = credentials('docker-hub-credentials')  // Single Docker credential
    BACKEND_IMAGE_NAME = "asp0217/employee-backend"
    FRONTEND_IMAGE_NAME = "asp0217/employee-frontend"
  }

  stages {
    stage('Checkout') {
      steps {
        // Checkout your repo
        git branch: 'main', url: 'https://github.com/atharv-pingle/devops-emp-app.git'
      }
    }

    stage('Build Backend') {
      steps {
        sh 'cd backend && ls -ltr'
        // Build the Go backend
        sh 'cd backend && go build -o app .'
      }
    }

    stage('Build Frontend') {
      steps {
        sh 'cd frontend && ls -ltr'
        // Build the React frontend
        sh 'cd frontend && npm install && npm run build'
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
          // Use the single Docker credentials to push the images
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
            git config user.email "your-email@example.com"
            git config user.name "Atharv Pingle"
            BUILD_NUMBER=${BUILD_NUMBER}
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
