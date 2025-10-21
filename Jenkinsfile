pipeline {
    agent any

    tools {
        maven 'Maven'
    }

    environment {
        DOCKER_HUB_CREDENTIALS = credentials('docker-hub-credentials')
        DOCKER_HUB_USERNAME = 'aryabhatt05'
        USER_SERVICE_IMAGE = "${DOCKER_HUB_USERNAME}/user-service"
        ORDER_SERVICE_IMAGE = "${DOCKER_HUB_USERNAME}/order-service"
        IMAGE_TAG = "${BUILD_NUMBER}"
        PATH = "/usr/local/bin:${env.PATH}"
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
                    sh 'mvn clean package -DskipTests'
                }
            }
        }

        stage('Build OrderService') {
            steps {
                echo 'Building OrderService...'
                dir('OrderService') {
                    sh 'mvn clean package -DskipTests'
                }
            }
        }

        stage('Test UserService') {
            steps {
                echo 'Testing UserService...'
                dir('UserService') {
                    sh 'mvn test'
                }
            }
        }

        stage('Test OrderService') {
            steps {
                echo 'Testing OrderService...'
                dir('OrderService') {
                    sh 'mvn test'
                }
            }
        }

        stage('Build Docker Images') {
            parallel {
                stage('Build UserService Image') {
                    steps {
                        echo 'Building UserService Docker image...'
                        dir('UserService') {
                            sh """
                                docker build -t ${USER_SERVICE_IMAGE}:${IMAGE_TAG} .
                                docker tag ${USER_SERVICE_IMAGE}:${IMAGE_TAG} ${USER_SERVICE_IMAGE}:latest
                            """
                        }
                    }
                }
                stage('Build OrderService Image') {
                    steps {
                        echo 'Building OrderService Docker image...'
                        dir('OrderService') {
                            sh """
                                docker build -t ${ORDER_SERVICE_IMAGE}:${IMAGE_TAG} .
                                docker tag ${ORDER_SERVICE_IMAGE}:${IMAGE_TAG} ${ORDER_SERVICE_IMAGE}:latest
                            """
                        }
                    }
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                echo 'Logging into Docker Hub...'
                sh 'echo $DOCKER_HUB_CREDENTIALS_PSW | docker login -u $DOCKER_HUB_CREDENTIALS_USR --password-stdin'

                echo 'Pushing UserService image...'
                sh """
                    docker push ${USER_SERVICE_IMAGE}:${IMAGE_TAG}
                    docker push ${USER_SERVICE_IMAGE}:latest
                """

                echo 'Pushing OrderService image...'
                sh """
                    docker push ${ORDER_SERVICE_IMAGE}:${IMAGE_TAG}
                    docker push ${ORDER_SERVICE_IMAGE}:latest
                """
            }
        }

        stage('Deploy Services') {
            steps {
                echo 'Deploying services...'
                sh '''
                    # Stop and remove existing containers
                    docker stop user-service order-service || true
                    docker rm user-service order-service || true

                    # Create network if not exists
                    docker network create microservices-network || true

                    # Deploy UserService
                    docker run -d \
                        --name user-service \
                        --network microservices-network \
                        -p 8081:8081 \
                        ${USER_SERVICE_IMAGE}:${IMAGE_TAG}

                    # Wait for UserService to be healthy
                    sleep 10

                    # Deploy OrderService
                    docker run -d \
                        --name order-service \
                        --network microservices-network \
                        -p 8082:8082 \
                        -e USERSERVICE_URL=http://user-service:8081 \
                        ${ORDER_SERVICE_IMAGE}:${IMAGE_TAG}
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                echo 'Verifying deployment...'
                sh '''
                    # Wait for services to start
                    sleep 15

                    # Check UserService health
                    echo "Checking UserService health..."
                    curl -f http://localhost:8081/api/users/health || exit 1

                    # Check OrderService health
                    echo "Checking OrderService health..."
                    curl -f http://localhost:8082/api/orders/health || exit 1

                    echo "All services are healthy!"
                '''
            }
        }
    }

    post {
        always {
            echo 'Cleaning up...'
            sh 'docker logout'
        }
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
            sh '''
                docker logs user-service || true
                docker logs order-service || true
            '''
        }
    }
}
