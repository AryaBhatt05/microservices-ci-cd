pipeline {
  agent any
  environment {
    DOCKER_HUB_CRED = 'docker-hub-cred'   // set in Jenkins credentials
    DOCKER_HUB_NAMESPACE = 'aryabhatt05'  // replace with your Docker Hub username/org
  }
  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build & Test - User Service') {
      steps {
        dir('user-service') {
          sh 'mvn -B -DskipTests=false clean package'
        }
      }
    }

    stage('Build & Test - Order Service') {
      steps {
        dir('order-service') {
          sh 'mvn -B -DskipTests=false clean package'
        }
      }
    }

    stage('Docker Build & Push') {
      steps {
        script {
          docker.withRegistry('', "${env.DOCKER_HUB_CRED}") {
            dir('user-service') {
              def userImg = docker.build("${env.DOCKER_HUB_NAMESPACE}/user-service:${env.BUILD_NUMBER}")
              userImg.push()
              userImg.push('latest')
            }
            dir('order-service') {
              def orderImg = docker.build("${env.DOCKER_HUB_NAMESPACE}/order-service:${env.BUILD_NUMBER}")
              orderImg.push()
              orderImg.push('latest')
            }
          }
        }
      }
    }

    stage('Deploy with docker-compose (local)') {
      steps {
        echo "Deploying with docker-compose on Jenkins host"
        sh '''
          docker-compose pull
          docker-compose up -d --remove-orphans
        '''
      }
    }

    // Alternative remote deploy (uncomment to use SSH)
    /*
    stage('Deploy to remote host via SSH') {
      steps {
        sshagent (credentials: ['ssh-deploy-cred']) {
          sh "scp docker-compose.yml user@<target-host>:/home/user/docker-compose.yml"
          sh "ssh -o StrictHostKeyChecking=no user@<target-host> 'docker-compose -f /home/user/docker-compose.yml pull && docker-compose -f /home/user/docker-compose.yml up -d'"
        }
      }
    }
    */
  }

  post {
    success {
      echo "Pipeline succeeded: ${env.BUILD_URL}"
    }
    failure {
      echo "Pipeline failed: ${env.BUILD_URL}"
    }
  }
}
