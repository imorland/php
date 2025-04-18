pipeline {
    agent none // We'll specify agents at the stage level
    options {
        skipDefaultCheckout(false)
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 120, unit: 'MINUTES')
    }
    environment {
        DOCKER_HUB_CREDENTIALS = credentials('docker-hub-credentials')
        DOCKER_BUILDKIT        = '1'
        DOCKER_CLI_EXPERIMENTAL = 'enabled'
        DOCKER_NAMESPACE       = 'ianmgg'
        PHP_VERSION            = '8.3'
        TAG_VERSION            = '83'
    }
    stages {
        stage('Prepare Workspace') {
            agent any
            steps {
                checkout scm
                sh 'echo $DOCKER_HUB_CREDENTIALS_PSW | docker login -u $DOCKER_HUB_CREDENTIALS_USR --password-stdin'
                sh 'docker version'
                sh 'docker buildx version'
                // Use tonistiigi/binfmt for better cross-platform support
                sh 'docker run --privileged --rm tonistiigi/binfmt --install all || docker run --rm --privileged multiarch/qemu-user-static --reset -p yes'
                stash includes: '**', name: 'source-code'
            }
        }
        
        // Build all images in parallel, maximizing executor usage
        stage('Build All Images') {
            parallel {
                stage('Build Apache AMD64') {
                    agent any
                    steps {
                        unstash 'source-code'
                        script {
                            // QEMU setup already done in Prepare Workspace stage
                            sh '''
                            docker buildx create --name PHPbuilder-apache-amd64-${BUILD_NUMBER} --use || true
                            docker buildx inspect PHPbuilder-apache-amd64-${BUILD_NUMBER} --bootstrap
                            '''
                            
                            sh """
                            docker buildx build . \
                              --progress=plain \
                              --platform linux/amd64 \
                              --file 8/${PHP_VERSION}/Dockerfile.apache \
                              --tag ${DOCKER_NAMESPACE}/php${TAG_VERSION}:latest-amd64 \
                              --push \
                              --no-cache \
                              --build-arg BUILDKIT_INLINE_CACHE=1
                            """
                        }
                    }
                    post {
                        always {
                            sh 'docker buildx rm PHPbuilder-apache-amd64-${BUILD_NUMBER} || true'
                            cleanWs()
                        }
                    }
                }
                
                stage('Build Apache ARM64') {
                    agent any
                    steps {
                        unstash 'source-code'
                        script {
                            sh 'docker run --rm --privileged multiarch/qemu-user-static --reset -p yes'
                            sh '''
                            docker buildx create --name PHPbuilder-apache-arm64-${BUILD_NUMBER} --use || true
                            docker buildx inspect PHPbuilder-apache-arm64-${BUILD_NUMBER} --bootstrap
                            '''
                            
                            sh """
                            docker buildx build . \
                              --progress=plain \
                              --platform linux/arm64 \
                              --file 8/${PHP_VERSION}/Dockerfile.apache \
                              --tag ${DOCKER_NAMESPACE}/php${TAG_VERSION}:latest-arm64 \
                              --push \
                              --no-cache \
                              --build-arg BUILDKIT_INLINE_CACHE=1
                            """
                        }
                    }
                    post {
                        always {
                            sh 'docker buildx rm PHPbuilder-apache-arm64-${BUILD_NUMBER} || true'
                            cleanWs()
                        }
                    }
                }
                
                stage('Build CLI AMD64') {
                    agent any
                    steps {
                        unstash 'source-code'
                        script {
                            sh 'docker run --rm --privileged multiarch/qemu-user-static --reset -p yes'
                            sh '''
                            docker buildx create --name PHPbuilder-cli-amd64-${BUILD_NUMBER} --use || true
                            docker buildx inspect PHPbuilder-cli-amd64-${BUILD_NUMBER} --bootstrap
                            '''
                            
                            sh """
                            docker buildx build . \
                              --progress=plain \
                              --platform linux/amd64 \
                              --file 8/${PHP_VERSION}/Dockerfile.cli \
                              --tag ${DOCKER_NAMESPACE}/php${TAG_VERSION}:cli-amd64 \
                              --push \
                              --no-cache \
                              --build-arg BUILDKIT_INLINE_CACHE=1
                            """
                        }
                    }
                    post {
                        always {
                            sh 'docker buildx rm PHPbuilder-cli-amd64-${BUILD_NUMBER} || true'
                            cleanWs()
                        }
                    }
                }
                
                stage('Build CLI ARM64') {
                    agent any
                    steps {
                        unstash 'source-code'
                        script {
                            sh 'docker run --rm --privileged multiarch/qemu-user-static --reset -p yes'
                            sh '''
                            docker buildx create --name PHPbuilder-cli-arm64-${BUILD_NUMBER} --use || true
                            docker buildx inspect PHPbuilder-cli-arm64-${BUILD_NUMBER} --bootstrap
                            '''
                            
                            sh """
                            docker buildx build . \
                              --progress=plain \
                              --platform linux/arm64 \
                              --file 8/${PHP_VERSION}/Dockerfile.cli \
                              --tag ${DOCKER_NAMESPACE}/php${TAG_VERSION}:cli-arm64 \
                              --push \
                              --no-cache \
                              --build-arg BUILDKIT_INLINE_CACHE=1
                            """
                        }
                    }
                    post {
                        always {
                            sh 'docker buildx rm PHPbuilder-cli-arm64-${BUILD_NUMBER} || true'
                            cleanWs()
                        }
                    }
                }
            }
        }
        
        // Create manifests to combine architecture-specific images
        stage('Create Base Manifests') {
            agent any
            steps {
                unstash 'source-code'
                script {
                    // Create manifest for Apache image
                    sh """
                    docker manifest create ${DOCKER_NAMESPACE}/php${TAG_VERSION}:latest \
                      --amend ${DOCKER_NAMESPACE}/php${TAG_VERSION}:latest-amd64 \
                      --amend ${DOCKER_NAMESPACE}/php${TAG_VERSION}:latest-arm64
                    docker manifest push ${DOCKER_NAMESPACE}/php${TAG_VERSION}:latest
                    """
                    
                    // Create manifest for CLI image
                    sh """
                    docker manifest create ${DOCKER_NAMESPACE}/php${TAG_VERSION}:cli \
                      --amend ${DOCKER_NAMESPACE}/php${TAG_VERSION}:cli-amd64 \
                      --amend ${DOCKER_NAMESPACE}/php${TAG_VERSION}:cli-arm64
                    docker manifest push ${DOCKER_NAMESPACE}/php${TAG_VERSION}:cli
                    """
                }
            }
            post {
                always {
                    cleanWs()
                }
            }
        }
        
        // Dev image builds after manifests are created
        stage('Build Dev Images') {
            parallel {
                stage('Build Dev AMD64') {
                    agent any
                    steps {
                        unstash 'source-code'
                        script {
                            sh 'docker run --rm --privileged multiarch/qemu-user-static --reset -p yes'
                            sh '''
                            docker buildx create --name PHPbuilder-dev-amd64-${BUILD_NUMBER} --use || true
                            docker buildx inspect PHPbuilder-dev-amd64-${BUILD_NUMBER} --bootstrap
                            '''
                            
                            sh """
                            docker buildx build . \
                              --progress=plain \
                              --platform linux/amd64 \
                              --file 8/${PHP_VERSION}/Dockerfile.apache.dev \
                              --tag ${DOCKER_NAMESPACE}/php${TAG_VERSION}:dev-amd64 \
                              --push \
                              --no-cache \
                              --build-arg BUILDKIT_INLINE_CACHE=1
                            """
                        }
                    }
                    post {
                        always {
                            sh 'docker buildx rm PHPbuilder-dev-amd64-${BUILD_NUMBER} || true'
                            cleanWs()
                        }
                    }
                }
                
                stage('Build Dev ARM64') {
                    agent any
                    steps {
                        unstash 'source-code'
                        script {
                            sh 'docker run --rm --privileged multiarch/qemu-user-static --reset -p yes'
                            sh '''
                            docker buildx create --name PHPbuilder-dev-arm64-${BUILD_NUMBER} --use || true
                            docker buildx inspect PHPbuilder-dev-arm64-${BUILD_NUMBER} --bootstrap
                            '''
                            
                            sh """
                            docker buildx build . \
                              --progress=plain \
                              --platform linux/arm64 \
                              --file 8/${PHP_VERSION}/Dockerfile.apache.dev \
                              --tag ${DOCKER_NAMESPACE}/php${TAG_VERSION}:dev-arm64 \
                              --push \
                              --no-cache \
                              --build-arg BUILDKIT_INLINE_CACHE=1
                            """
                        }
                    }
                    post {
                        always {
                            sh 'docker buildx rm PHPbuilder-dev-arm64-${BUILD_NUMBER} || true'
                            cleanWs()
                        }
                    }
                }
            }
        }
        
        // Create manifest for Dev image
        stage('Create Dev Manifest') {
            agent any
            steps {
                unstash 'source-code'
                script {
                    sh """
                    docker manifest create ${DOCKER_NAMESPACE}/php${TAG_VERSION}:dev \
                      --amend ${DOCKER_NAMESPACE}/php${TAG_VERSION}:dev-amd64 \
                      --amend ${DOCKER_NAMESPACE}/php${TAG_VERSION}:dev-arm64
                    docker manifest push ${DOCKER_NAMESPACE}/php${TAG_VERSION}:dev
                    """
                }
            }
            post {
                always {
                    cleanWs()
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
        always {
            node(null) {
                // Final cleanup
                sh 'docker buildx prune -f || true'
            }
        }
    }
}
