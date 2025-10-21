// Jenkinsfile - cross-platform, build -> push -> deploy
pipeline {
  agent any

  tools {
    // Ensure a Maven installation named "Maven" exists in Jenkins Global Tool Configuration
    maven 'Maven'
  }

  environment {
    // Ensure this credential ID exists in Jenkins (Username with password: username = DockerHub user, password = token)
    DOCKER_HUB_CREDENTIALS = credentials('docker-hub-credentials')
    DOCKER_HUB_USERNAME = 'aryabhatt05'
    USER_SERVICE_IMAGE = "${DOCKER_HUB_USERNAME}/user-service"
    ORDER_SERVICE_IMAGE = "${DOCKER_HUB_USERNAME}/order-service"
    IMAGE_TAG = "${BUILD_NUMBER}"
    COMPOSE_FILE = 'docker-compose.yml'
  }

  options {
    timestamps()
    ansiColor('xterm')
    timeout(time: 45, unit: 'MINUTES')
    buildDiscarder(logRotator(numToKeepStr: '10'))
  }

  stages {
    stage('Checkout') {
      steps {
        echo 'Checking out source code...'
        checkout scm
      }
    }

    stage('Build & Package') {
      parallel {
        stage('Build - user-service') {
          when { expression { fileExists('user-service/pom.xml') } }
          steps {
            dir('user-service') {
              script {
                if (isUnix()) {
                  sh 'mvn -B clean package -DskipTests'
                } else {
                  bat 'mvn -B clean package -DskipTests'
                }
              }
            }
          }
        }

        stage('Build - order-service') {
          when { expression { fileExists('order-service/pom.xml') } }
          steps {
            dir('order-service') {
              script {
                if (isUnix()) {
                  sh 'mvn -B clean package -DskipTests'
                } else {
                  bat 'mvn -B clean package -DskipTests'
                }
              }
            }
          }
        }
      }
    }

    stage('Test') {
      parallel {
        stage('Test - user-service') {
          when { expression { fileExists('user-service/pom.xml') } }
          steps {
            dir('user-service') {
              script {
                if (isUnix()) { sh 'mvn test -q' } else { bat 'mvn test -q' }
              }
            }
          }
        }

        stage('Test - order-service') {
          when { expression { fileExists('order-service/pom.xml') } }
          steps {
            dir('order-service') {
              script {
                if (isUnix()) { sh 'mvn test -q' } else { bat 'mvn test -q' }
              }
            }
          }
        }
      }
    }

    stage('Docker Build & Tag') {
      when { anyOf { expression { fileExists('user-service/Dockerfile') }; expression { fileExists('order-service/Dockerfile') } } }
      steps {
        script {
          // Login handled in Push stage with withCredentials
          if (fileExists('user-service/Dockerfile')) {
            dir('user-service') {
              if (isUnix()) {
                sh "docker build -t ${USER_SERVICE_IMAGE}:${IMAGE_TAG} ."
                sh "docker tag ${USER_SERVICE_IMAGE}:${IMAGE_TAG} ${USER_SERVICE_IMAGE}:latest"
              } else {
                bat "docker build -t ${USER_SERVICE_IMAGE}:${IMAGE_TAG} ."
                bat "docker tag ${USER_SERVICE_IMAGE}:${IMAGE_TAG} ${USER_SERVICE_IMAGE}:latest"
              }
            }
          }
          if (fileExists('order-service/Dockerfile')) {
            dir('order-service') {
              if (isUnix()) {
                sh "docker build -t ${ORDER_SERVICE_IMAGE}:${IMAGE_TAG} ."
                sh "docker tag ${ORDER_SERVICE_IMAGE}:${IMAGE_TAG} ${ORDER_SERVICE_IMAGE}:latest"
              } else {
                bat "docker build -t ${ORDER_SERVICE_IMAGE}:${IMAGE_TAG} ."
                bat "docker tag ${ORDER_SERVICE_IMAGE}:${IMAGE_TAG} ${ORDER_SERVICE_IMAGE}:latest"
              }
            }
          }
        }
      }
    }

    stage('Push to Docker Hub') {
      when { expression { return fileExists('user-service/Dockerfile') || fileExists('order-service/Dockerfile') } }
      steps {
        script {
          // credentials(...) in environment created DOCKER_HUB_CREDENTIALS_USR and _PSW, but using withCredentials is explicit and safer
          withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
            if (isUnix()) {
              sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
            } else {
              // Windows: use PowerShell style piping for password (works with Docker Desktop)
              bat 'echo %DOCKER_PASS% | docker login -u %DOCKER_USER% --password-stdin'
            }

            // push images if they exist locally
            if (fileExists('user-service/Dockerfile')) {
              if (isUnix()) {
                sh "docker push ${USER_SERVICE_IMAGE}:${IMAGE_TAG}"
                sh "docker push ${USER_SERVICE_IMAGE}:latest"
              } else {
                bat "docker push ${USER_SERVICE_IMAGE}:${IMAGE_TAG}"
                bat "docker push ${USER_SERVICE_IMAGE}:latest"
              }
            }
            if (fileExists('order-service/Dockerfile')) {
              if (isUnix()) {
                sh "docker push ${ORDER_SERVICE_IMAGE}:${IMAGE_TAG}"
                sh "docker push ${ORDER_SERVICE_IMAGE}:latest"
              } else {
                bat "docker push ${ORDER_SERVICE_IMAGE}:${IMAGE_TAG}"
                bat "docker push ${ORDER_SERVICE_IMAGE}:latest"
              }
            }
          } // withCredentials
        } // script
      } // steps
    } // stage

    stage('Deploy - local docker host') {
      steps {
        script {
          echo "Deploying containers (local docker host)..."
          if (isUnix()) {
            sh """
              docker rm -f user-service order-service || true
              docker network create microservices-network || true
              docker run -d --name user-service --network microservices-network -p 8081:8081 ${USER_SERVICE_IMAGE}:${IMAGE_TAG}
              sleep 8
              docker run -d --name order-service --network microservices-network -p 8082:8082 -e USERSERVICE_URL=http://user-service:8081 ${ORDER_SERVICE_IMAGE}:${IMAGE_TAG}
            """
          } else {
            bat """
              docker rm -f user-service order-service || echo rm-f-failed
              docker network create microservices-network || echo net-exists
              docker run -d --name user-service --network microservices-network -p 8081:8081 ${USER_SERVICE_IMAGE}:${IMAGE_TAG}
              timeout /T 8 /NOBREAK
              docker run -d --name order-service --network microservices-network -p 8082:8082 -e USERSERVICE_URL=http://user-service:8081 ${ORDER_SERVICE_IMAGE}:${IMAGE_TAG}
            """
          }
        }
      }
    }

    stage('Verify Deployment') {
      steps {
        script {
          def check = { String url ->
            if (isUnix()) {
              return sh(script: "curl -s -o /dev/null -w '%{http_code}' ${url} || echo 000", returnStdout: true).trim()
            } else {
              return bat(script: "powershell -Command \"try { (Invoke-WebRequest -UseBasicParsing -Uri '${url}' -TimeoutSec 5).StatusCode } catch { Write-Output '000' }\"", returnStdout: true).trim()
            }
          }

          // try a few endpoints / fallbacks
          def targets = [
            'http://localhost:8081/actuator/health',
            'http://localhost:8081/api/users/health',
            'http://localhost:8081/api/users/hello'
          ]
          def ok = false
          for (t in targets) {
            def code = check(t)
            echo "Checked ${t} -> ${code}"
            if (code.startsWith('2')) { ok = true; break }
          }
          if (!ok) { error "UserService health check failed." }

          targets = [
            'http://localhost:8082/actuator/health',
            'http://localhost:8082/api/orders/health',
            'http://localhost:8082/api/orders/hello'
          ]
          ok = false
          for (t in targets) {
            def code = check(t)
            echo "Checked ${t} -> ${code}"
            if (code.startsWith('2')) { ok = true; break }
          }
          if (!ok) { error "OrderService health check failed." }

          echo "Both services responded successfully."
        } // script
      } // steps
    } // stage
  } // stages

  post {
    always {
      echo 'Pipeline finished - attempting docker logout'
      script {
        if (isUnix()) { sh 'docker logout || true' } else { bat 'docker logout || echo logout-failed' }
      }
    }
    success { echo 'Pipeline completed successfully!' }
    failure {
      echo 'Pipeline failed - printing container logs (if any)'
      script {
        if (isUnix()) {
          sh 'docker logs user-service || true; docker logs order-service || true'
        } else {
          bat 'docker logs user-service || echo no-logs & docker logs order-service || echo no-logs'
        }
      }
    }
  }
}
