pipeline {
    agent any
    environment {
        DOCKER_HUB_CREDENTIALS = credentials('docker-hub-credentials')
        DOCKER_BUILDKIT = "1"
        DOCKER_CLI_EXPERIMENTAL = "enabled"
        DOCKER_NAMESPACE = "ianmgg"
    }
    stages {
        stage('Test Docker') {
            steps {
                sh 'docker --version'
            }
        }
        stage('Set up QEMU and Docker Buildx') {
            steps {
                script {
                    sh 'docker run --rm --privileged multiarch/qemu-user-static --reset -p yes'
                    // Set up Docker Buildx if not already available
                    sh '''
                    docker buildx create --name phpbuilder --use || true
                    docker buildx inspect phpbuilder --bootstrap
                    '''
                }
            }
        }
        stage('Build Images in Parallel') {
            matrix {
                axes {
                    axis {
                        name 'PHP_VERSION'
                        values '8.3', '8.2', '8.1'
                    }
                }
                stages {
                    stage('Build Images') { // Static stage name
                        steps {
                            script {
                                echo "Building images for PHP ${PHP_VERSION}" // Log PHP version
                                def tagVersion = PHP_VERSION.replace('.', '')

                                // Login to Docker Hub
                                sh 'echo $DOCKER_HUB_CREDENTIALS_PSW | docker login -u $DOCKER_HUB_CREDENTIALS_USR --password-stdin'

                                // Nested stages
                                stage('Build Latest Image') {
                                    sh """
                                    docker buildx build . \
                                      --platform linux/amd64,linux/arm64 \
                                      --file 8/${PHP_VERSION}/Dockerfile.apache \
                                      --tag ${DOCKER_NAMESPACE}/php${tagVersion}:latest \
                                      --push
                                    """
                                }
                                stage('Build CLI Image') {
                                    sh """
                                    docker buildx build . \
                                      --platform linux/amd64,linux/arm64 \
                                      --file 8/${PHP_VERSION}/Dockerfile.cli \
                                      --tag ${DOCKER_NAMESPACE}/php${tagVersion}:cli \
                                      --push
                                    """
                                }
                                stage('Build Dev Image') {
                                    sh """
                                    docker buildx build . \
                                      --platform linux/amd64,linux/arm64 \
                                      --file 8/${PHP_VERSION}/Dockerfile.apache.dev \
                                      --tag ${DOCKER_NAMESPACE}/php${tagVersion}:dev \
                                      --push
                                    """
                                }
                            }
                        }
                    }
                }
            }
        }
    }
    post {
        always {
            script {
                // Clean up builder
                sh 'docker buildx rm phpbuilder || true'
            }
        }
        success {
            echo 'All Docker images successfully built and pushed to Docker Hub!'
        }
        failure {
            echo 'Build or push failed.'
        }
    }
}
