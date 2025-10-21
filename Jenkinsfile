// Jenkinsfile - cross-platform (Windows + Linux) with optional image build & remote deploy
pipeline {
  agent any

  environment {
    DOCKER_HUB_CRED = 'docker-hub-cred'      // Docker Hub creds in Jenkins (username + token)
    DOCKER_HUB_NAMESPACE = 'aryabhatt05'
    COMPOSE_FILE = 'docker-compose.yml'
    // Toggle these as build parameters if you want them configurable in Jenkins job
    DO_BUILD_IMAGES = 'true'                 // "true" to build & push images, "false" to skip
    DEPLOY_REMOTE = 'false'                  // "true" to copy compose and run on remote host
    DEPLOY_USER = 'ubuntu'                   // remote deploy user (if DEPLOY_REMOTE=true)
    DEPLOY_HOST = '1.2.3.4'                  // remote host IP or hostname
  }

  options {
    timestamps()
    ansiColor('xterm')
    // keep last 5 builds
    buildDiscarder(logRotator(numToKeepStr: '5'))
  }

  stages {
    stage('Checkout') {
      steps {
        script {
          if (env.GIT_CRED && env.GIT_CRED != '') {
            checkout([$class: 'GitSCM',
                      branches: [[name: '*/main']],
                      userRemoteConfigs: [[url: '<git-repo-url>', credentialsId: env.GIT_CRED]]])
          } else {
            checkout([$class: 'GitSCM',
                      branches: [[name: '*/main']],
                      userRemoteConfigs: [[url: '<git-repo-url>']]])
          }
        }
      }
    }

    stage('Build & Test') {
      parallel {
        stage('Build - user-service') {
          when { expression { fileExists('user-service/pom.xml') } }
          steps {
            dir('user-service') {
              script {
                if (isUnix()) {
                  sh 'mvn -B clean package -DskipTests=false'
                } else {
                  bat 'mvn -B clean package -DskipTests=false'
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
                  sh 'mvn -B clean package -DskipTests=false'
                } else {
                  bat 'mvn -B clean package -DskipTests=false'
                }
              }
            }
          }
        }
      }
    }

    stage('Docker Build & Push (optional)') {
      when { expression { return env.DO_BUILD_IMAGES == 'true' } }
      steps {
        script {
          docker.withRegistry('', env.DOCKER_HUB_CRED) {
            if (fileExists('user-service/Dockerfile')) {
              dir('user-service') {
                if (isUnix()) {
                  sh "docker build -t ${DOCKER_HUB_NAMESPACE}/user-service:${BUILD_NUMBER} ."
                  sh "docker push ${DOCKER_HUB_NAMESPACE}/user-service:${BUILD_NUMBER}"
                  sh "docker tag ${DOCKER_HUB_NAMESPACE}/user-service:${BUILD_NUMBER} ${DOCKER_HUB_NAMESPACE}/user-service:latest"
                  sh "docker push ${DOCKER_HUB_NAMESPACE}/user-service:latest"
                } else {
                  bat "docker build -t ${DOCKER_HUB_NAMESPACE}/user-service:${BUILD_NUMBER} ."
                  bat "docker push ${DOCKER_HUB_NAMESPACE}/user-service:${BUILD_NUMBER}"
                  bat "docker tag ${DOCKER_HUB_NAMESPACE}/user-service:${BUILD_NUMBER} ${DOCKER_HUB_NAMESPACE}/user-service:latest"
                  bat "docker push ${DOCKER_HUB_NAMESPACE}/user-service:latest"
                }
              }
            }

            if (fileExists('order-service/Dockerfile')) {
              dir('order-service') {
                if (isUnix()) {
                  sh "docker build -t ${DOCKER_HUB_NAMESPACE}/order-service:${BUILD_NUMBER} ."
                  sh "docker push ${DOCKER_HUB_NAMESPACE}/order-service:${BUILD_NUMBER}"
                  sh "docker tag ${DOCKER_HUB_NAMESPACE}/order-service:${BUILD_NUMBER} ${DOCKER_HUB_NAMESPACE}/order-service:latest"
                  sh "docker push ${DOCKER_HUB_NAMESPACE}/order-service:latest"
                } else {
                  bat "docker build -t ${DOCKER_HUB_NAMESPACE}/order-service:${BUILD_NUMBER} ."
                  bat "docker push ${DOCKER_HUB_NAMESPACE}/order-service:${BUILD_NUMBER}"
                  bat "docker tag ${DOCKER_HUB_NAMESPACE}/order-service:${BUILD_NUMBER} ${DOCKER_HUB_NAMESPACE}/order-service:latest"
                  bat "docker push ${DOCKER_HUB_NAMESPACE}/order-service:latest"
                }
              }
            }
          } // withRegistry
        } // script
      } // steps
    } // stage

    stage('Deploy - Local docker-compose') {
      when { expression { return env.DEPLOY_REMOTE != 'true' } }
      steps {
        script {
          if (isUnix()) {
            sh "docker-compose -f ${COMPOSE_FILE} pull || true"
            sh "docker-compose -f ${COMPOSE_FILE} up -d --remove-orphans"
          } else {
            bat "docker-compose -f ${COMPOSE_FILE} pull || (echo pull-failed)"
            bat "docker-compose -f ${COMPOSE_FILE} up -d --remove-orphans"
          }
        }
      }
    }

    stage('Deploy - Remote via SSH (optional)') {
      when { expression { return env.DEPLOY_REMOTE == 'true' } }
      steps {
        sshagent (credentials: [env.SSH_DEPLOY_CRED]) {
          script {
            def target = "${DEPLOY_USER}@${DEPLOY_HOST}"
            // copy compose file
            if (isUnix()) {
              sh "scp -o StrictHostKeyChecking=no ${COMPOSE_FILE} ${target}:/home/${DEPLOY_USER}/${COMPOSE_FILE}"
              sh "ssh -o StrictHostKeyChecking=no ${target} 'docker-compose -f /home/${DEPLOY_USER}/${COMPOSE_FILE} pull && docker-compose -f /home/${DEPLOY_USER}/${COMPOSE_FILE} up -d --remove-orphans'"
            } else {
              bat "pscp -batch ${COMPOSE_FILE} ${target}:/home/${DEPLOY_USER}/${COMPOSE_FILE}"
              bat "plink -batch ${target} \"docker-compose -f /home/${DEPLOY_USER}/${COMPOSE_FILE} pull && docker-compose -f /home/${DEPLOY_USER}/${COMPOSE_FILE} up -d --remove-orphans\""
            }
          }
        }
      }
    }

  } // stages

  post {
    success {
      echo "Pipeline succeeded: ${env.BUILD_URL}"
    }
    failure {
      echo "Pipeline failed: ${env.BUILD_URL}"
    }
    always {
      script {
        // Non-fatal cleanup
        if (isUnix()) {
          sh 'docker image prune -af || true'
        } else {
          bat 'docker image prune -af || exit /b 0'
        }
      }
    }
  }
}
