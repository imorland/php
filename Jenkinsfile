pipeline {
    agent any
    options {
        skipDefaultCheckout(false)
    }
    environment {
        DOCKER_HUB_CREDENTIALS = credentials('docker-hub-credentials')
        WATCHTOWER_TOKEN       = credentials('watchtower-token')
        DOCKER_BUILDKIT        = '1'
        DOCKER_CLI_EXPERIMENTAL = 'enabled'
        DOCKER_NAMESPACE       = 'ianmgg'
        PHP_VERSION            = '8.3'
        TAG_VERSION            = '83'
    }
    stages {
        stage('Setup') {
            steps {
                script {
                    // Login to Docker Hub once at the beginning
                    sh 'echo $DOCKER_HUB_CREDENTIALS_PSW | docker login -u $DOCKER_HUB_CREDENTIALS_USR --password-stdin'
                }
            }
        }
        stage('Build Images in Parallel') {
            parallel {
                stage('Build Apache Image') {
                    agent any
                    steps {
                        script {
                            // Set up QEMU and Buildx on this agent
                            sh 'docker run --rm --privileged multiarch/qemu-user-static --reset -p yes'
                            sh '''
                            docker buildx create --name PHPbuilder-apache --use || true
                            docker buildx inspect PHPbuilder-apache --bootstrap
                            '''
                            
                            // Build and push the Apache image
                            sh """
                            docker buildx build . \
                              --progress=plain \
                              --platform linux/amd64,linux/arm64 \
                              --file 8/${PHP_VERSION}/Dockerfile.apache \
                              --tag ${DOCKER_NAMESPACE}/php${TAG_VERSION}:latest \
                              --push
                            """
                        }
                    }
                    post {
                        always {
                            sh 'docker buildx rm PHPbuilder-apache || true'
                        }
                    }
                }
                
                stage('Build CLI Image') {
                    agent any
                    steps {
                        script {
                            // Set up QEMU and Buildx on this agent
                            sh 'docker run --rm --privileged multiarch/qemu-user-static --reset -p yes'
                            sh '''
                            docker buildx create --name PHPbuilder-cli --use || true
                            docker buildx inspect PHPbuilder-cli --bootstrap
                            '''
                            
                            // Build and push the CLI image
                            sh """
                            docker buildx build . \
                              --progress=plain \
                              --platform linux/amd64,linux/arm64 \
                              --file 8/${PHP_VERSION}/Dockerfile.cli \
                              --tag ${DOCKER_NAMESPACE}/php${TAG_VERSION}:cli \
                              --push
                            """
                        }
                    }
                    post {
                        always {
                            sh 'docker buildx rm PHPbuilder-cli || true'
                        }
                    }
                }
            }
        }
        
        // Dev image depends on Apache image, so it runs after the parallel stage
        stage('Build Dev Image') {
            steps {
                script {
                    // Set up QEMU and Buildx
                    sh 'docker run --rm --privileged multiarch/qemu-user-static --reset -p yes'
                    sh '''
                    docker buildx create --name PHPbuilder-dev --use || true
                    docker buildx inspect PHPbuilder-dev --bootstrap
                    '''
                    
                    // Build and push the Dev image
                    sh """
                    docker buildx build . \
                      --progress=plain \
                      --platform linux/amd64,linux/arm64 \
                      --file 8/${PHP_VERSION}/Dockerfile.apache.dev \
                      --tag ${DOCKER_NAMESPACE}/php${TAG_VERSION}:dev \
                      --push
                    """
                }
            }
            post {
                always {
                    sh 'docker buildx rm PHPbuilder-dev || true'
                }
            }
        }
        
        stage('Schedule Watchtower Update') {
            steps {
                script {
                    sh '''
                    docker run -d --rm \
                      busybox:1.35 \
                      sh -c "sleep 60 && \
                        wget -qO- \\
                          --header 'Authorization: Bearer $WATCHTOWER_TOKEN' \\
                          http://host.docker.internal:8081/v1/update"
                    '''
                }
            }
        }
    }
    post {
        success {
            echo "Successfully built and pushed PHP ${PHP_VERSION} Docker images"
        }
        failure {
            echo "Failed to build PHP ${PHP_VERSION} Docker images"
        }
    }
}
