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
                    stage("Build Latest Image for PHP ${PHP_VERSION}") {
                        steps {
                            script {
                                def tagVersion = PHP_VERSION.replace('.', '')

                                // Login to Docker Hub
                                sh 'echo $DOCKER_HUB_CREDENTIALS_PSW | docker login -u $DOCKER_HUB_CREDENTIALS_USR --password-stdin'
                                
                                // Build Apache image (latest tag)
                                def apacheImage = "${DOCKER_NAMESPACE}/php${tagVersion}:latest"
                                sh """
                                docker buildx build . \
                                  --platform linux/amd64,linux/arm64 \
                                  --file 8/${PHP_VERSION}/Dockerfile.apache \
                                  --tag ${apacheImage} \
                                  --push
                                """
                            }
                        }
                    }
                    stage("Build CLI Image for PHP ${PHP_VERSION}") {
                        steps {
                            script {
                                def tagVersion = PHP_VERSION.replace('.', '')

                                // Build CLI image
                                def cliImage = "${DOCKER_NAMESPACE}/php${tagVersion}:cli"
                                sh """
                                docker buildx build . \
                                  --platform linux/amd64,linux/arm64 \
                                  --file 8/${PHP_VERSION}/Dockerfile.cli \
                                  --tag ${cliImage} \
                                  --push
                                """
                            }
                        }
                    }
                    stage("Build Dev Image for PHP ${PHP_VERSION}") {
                        steps {
                            script {
                                def tagVersion = PHP_VERSION.replace('.', '')

                                // Build Dev image
                                def devImage = "${DOCKER_NAMESPACE}/php${tagVersion}:dev"
                                sh """
                                docker buildx build . \
                                  --platform linux/amd64,linux/arm64 \
                                  --file 8/${PHP_VERSION}/Dockerfile.apache.dev \
                                  --tag ${devImage} \
                                  --push
                                """
                            }
                        }
                    }
                }
                when {
                    beforeAgent true
                    expression { env.PHP_VERSION != null }
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
