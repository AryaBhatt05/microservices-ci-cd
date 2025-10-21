pipeline {
    agent any

    tools {
        maven 'Maven'
    }

environment {
    DOCKER_HUB_CREDENTIALS = credentials('docker-hub-cred')
    DOCKER_HUB_USERNAME = 'aryabhatt05'
    USER_SERVICE_IMAGE = "${DOCKER_HUB_USERNAME}/user-serice"
    ORDER_SERVICE_IMAGE = "${DOCKER_HUB_USERNAME}/order-service"
    IMAGE_TAG = "${BUILD_NUMBER}"
}


    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                checkout scm
            }
        }

        stage('Build UserService') {
            steps {
                echo 'Building UserService...'
                dir('UserService') {
                    bat 'mvn clean package -DskipTests'
                }
            }
        }

        stage('Build OrderService') {
            steps {
                echo 'Building OrderService...'
                dir('OrderService') {
                    bat 'mvn clean package -DskipTests'
                }
            }
        }

        stage('Test Services') {
            steps {
                echo 'Running tests...'
                dir('UserService') {
                    bat 'mvn test'
                }
                dir('OrderService') {
                    bat 'mvn test'
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                echo 'Building Docker images...'
                bat """
                    docker build -t ${USER_SERVICE_IMAGE}:${IMAGE_TAG} ./UserService
                    docker build -t ${ORDER_SERVICE_IMAGE}:${IMAGE_TAG} ./OrderService
                    docker tag ${USER_SERVICE_IMAGE}:${IMAGE_TAG} ${USER_SERVICE_IMAGE}:latest
                    docker tag ${ORDER_SERVICE_IMAGE}:${IMAGE_TAG} ${ORDER_SERVICE_IMAGE}:latest
                """
            }
        }

        stage('Push to Docker Hub') {
            steps {
                echo 'Logging into Docker Hub...'
                bat """
                    echo ${DOCKER_HUB_CREDENTIALS_PSW} | docker login -u ${DOCKER_HUB_CREDENTIALS_USR} --password-stdin
                    docker push ${USER_SERVICE_IMAGE}:${IMAGE_TAG}
                    docker push ${USER_SERVICE_IMAGE}:latest
                    docker push ${ORDER_SERVICE_IMAGE}:${IMAGE_TAG}
                    docker push ${ORDER_SERVICE_IMAGE}:latest
                """
            }
        }

        stage('Deploy Containers') {
            steps {
                echo 'Deploying containers...'
                bat """
                    docker stop user-service order-service || exit 0
                    docker rm user-service order-service || exit 0
                    docker network create microservices-network || exit 0

                    docker run -d --name user-service --network microservices-network -p 8081:8081 ${USER_SERVICE_IMAGE}:${IMAGE_TAG}
                    timeout /t 10
                    docker run -d --name order-service --network microservices-network -p 8082:8082 -e USERSERVICE_URL=http://user-service:8081 ${ORDER_SERVICE_IMAGE}:${IMAGE_TAG}
                """
            }
        }

        stage('Verify Deployment') {
            steps {
                echo 'Verifying deployment...'
                bat """
                    timeout /t 15
                    curl -f http://localhost:8081/api/users/health
                    curl -f http://localhost:8082/api/orders/health
                """
            }
        }
    }

    post {
        always {
            script {
                echo 'Pipeline finished - attempting docker logout'
                bat 'docker logout'
            }
        }
        success {
            echo '✅ Pipeline completed successfully!'
        }
        failure {
            script {
                echo '❌ Pipeline failed - printing container logs (if any)'
                bat 'docker logs user-service || exit 0'
                bat 'docker logs order-service || exit 0'
            }
        }
    }
}
