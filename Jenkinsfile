pipeline {
    agent any

    tools {
        maven 'Maven'
    }

    environment {
        DOCKER_HUB_CREDENTIALS = credentials('docker-hub-cred')
        DOCKER_HUB_USERNAME = 'aryabhatt05'
        USER_SERVICE_IMAGE = "${DOCKER_HUB_USERNAME}/user-service"
        ORDER_SERVICE_IMAGE = "${DOCKER_HUB_USERNAME}/order-service"
        IMAGE_TAG = "${BUILD_NUMBER}"
        PATH = "C:\\Program Files\\Docker\\Docker\\resources\\bin;${env.PATH}"
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
                echo 'Building user-service...'
                dir('user-service') {
                    bat 'mvn clean package -DskipTests'
                }
            }
        }

        stage('Build OrderService') {
            steps {
                echo 'Building order-service...'
                dir('order-service') {
                    bat 'mvn clean package -DskipTests'
                }
            }
        }

        stage('Test UserService') {
            steps {
                echo 'Running tests for user-service...'
                dir('user-service') {
                    bat 'mvn test'
                }
            }
        }

        stage('Test OrderService') {
            steps {
                echo 'Running tests for order-service...'
                dir('order-service') {
                    bat 'mvn test'
                }
            }
        }

        stage('Build Docker Images') {
            parallel {
                stage('Build user-service Image') {
                    steps {
                        echo 'Building Docker image for user-service...'
                        dir('user-service') {
                            bat """
                                docker build -t ${USER_SERVICE_IMAGE}:${IMAGE_TAG} .
                                docker tag ${USER_SERVICE_IMAGE}:${IMAGE_TAG} ${USER_SERVICE_IMAGE}:latest
                            """
                        }
                    }
                }
                stage('Build order-service Image') {
                    steps {
                        echo 'Building Docker image for order-service...'
                        dir('order-service') {
                            bat """
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
                echo 'Logging in and pushing Docker images to Docker Hub...'
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
                echo 'Deploying microservices containers...'
                bat """
                    docker stop user-service order-service || exit 0
                    docker rm user-service order-service || exit 0
                    docker network create microservices-network || exit 0

                    docker run -d --name user-service --network microservices-network -p 8081:8081 ${USER_SERVICE_IMAGE}:${IMAGE_TAG}
                    ping 127.0.0.1 -n 11 > nul

                    docker run -d --name order-service --network microservices-network -p 8082:8082 -e USERSERVICE_URL=http://user-service:8081 ${ORDER_SERVICE_IMAGE}:${IMAGE_TAG}
                    ping 127.0.0.1 -n 11 > nul
                """
            }
        }

        stage('Verify Deployment') {
            steps {
                echo 'Verifying deployed microservices...'

                bat """
                    REM ---- Retry UserService health check ----
                    set RETRIES=6
                    set WAIT=5
                    :check_user
                    curl -f http://localhost:8081/api/users/health
                    if %ERRORLEVEL% neq 0 (
                        echo UserService not ready, waiting %WAIT% seconds...
                        ping 127.0.0.1 -n %WAIT% > nul
                        set /a RETRIES-=1
                        if %RETRIES% gtr 0 goto check_user
                        echo UserService failed to start!
                        exit /b 1
                    )
                    echo UserService is healthy!

                    REM ---- Retry OrderService health check ----
                    set RETRIES=6
                    :check_order
                    curl -f http://localhost:8082/api/orders/health
                    if %ERRORLEVEL% neq 0 (
                        echo OrderService not ready, waiting %WAIT% seconds...
                        ping 127.0.0.1 -n %WAIT% > nul
                        set /a RETRIES-=1
                        if %RETRIES% gtr 0 goto check_order
                        echo OrderService failed to start!
                        exit /b 1
                    )
                    echo OrderService is healthy!
                """
            }
        }
    }

    post {
        always {
            echo 'Cleaning up Docker session...'
            bat 'docker logout'
        }
        success {
            echo '✅ Pipeline completed successfully! Microservices deployed.'
        }
        failure {
            echo '❌ Pipeline failed. Fetching container logs...'
            bat 'docker logs user-service || exit 0'
            bat 'docker logs order-service || exit 0'
        }
    }
}
