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
        DISCORD_WEBHOOK = credentials('discord-webhook-url')
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
                // Send build started notification
                discordSend webhookURL: "${DISCORD_WEBHOOK}", 
                           title: "Build Started: PHP ${PHP_VERSION} Docker Images", 
                           description: "Building PHP ${PHP_VERSION} Docker images (Apache, CLI, Dev) for amd64 and arm64 platforms",
                           link: env.BUILD_URL,
                           result: "INFO",
                           footer: "Build #${BUILD_NUMBER}"
                
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
                        success {
                            discordSend webhookURL: "${DISCORD_WEBHOOK}", 
                                       title: "Apache AMD64 Build Successful", 
                                       description: "Successfully built and pushed ${DOCKER_NAMESPACE}/php${TAG_VERSION}:latest-amd64",
                                       link: env.BUILD_URL,
                                       result: "SUCCESS"
                        }
                        failure {
                            discordSend webhookURL: "${DISCORD_WEBHOOK}", 
                                       title: "Apache AMD64 Build Failed", 
                                       description: "Failed to build ${DOCKER_NAMESPACE}/php${TAG_VERSION}:latest-amd64",
                                       link: env.BUILD_URL,
                                       result: "FAILURE"
                        }
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
                        success {
                            discordSend webhookURL: "${DISCORD_WEBHOOK}", 
                                       title: "Apache ARM64 Build Successful", 
                                       description: "Successfully built and pushed ${DOCKER_NAMESPACE}/php${TAG_VERSION}:latest-arm64",
                                       link: env.BUILD_URL,
                                       result: "SUCCESS"
                        }
                        failure {
                            discordSend webhookURL: "${DISCORD_WEBHOOK}", 
                                       title: "Apache ARM64 Build Failed", 
                                       description: "Failed to build ${DOCKER_NAMESPACE}/php${TAG_VERSION}:latest-arm64",
                                       link: env.BUILD_URL,
                                       result: "FAILURE"
                        }
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
                        success {
                            discordSend webhookURL: "${DISCORD_WEBHOOK}", 
                                       title: "CLI AMD64 Build Successful", 
                                       description: "Successfully built and pushed ${DOCKER_NAMESPACE}/php${TAG_VERSION}:cli-amd64",
                                       link: env.BUILD_URL,
                                       result: "SUCCESS"
                        }
                        failure {
                            discordSend webhookURL: "${DISCORD_WEBHOOK}", 
                                       title: "CLI AMD64 Build Failed", 
                                       description: "Failed to build ${DOCKER_NAMESPACE}/php${TAG_VERSION}:cli-amd64",
                                       link: env.BUILD_URL,
                                       result: "FAILURE"
                        }
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
                        success {
                            discordSend webhookURL: "${DISCORD_WEBHOOK}", 
                                       title: "CLI ARM64 Build Successful", 
                                       description: "Successfully built and pushed ${DOCKER_NAMESPACE}/php${TAG_VERSION}:cli-arm64",
                                       link: env.BUILD_URL,
                                       result: "SUCCESS"
                        }
                        failure {
                            discordSend webhookURL: "${DISCORD_WEBHOOK}", 
                                       title: "CLI ARM64 Build Failed", 
                                       description: "Failed to build ${DOCKER_NAMESPACE}/php${TAG_VERSION}:cli-arm64",
                                       link: env.BUILD_URL,
                                       result: "FAILURE"
                        }
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
                success {
                    discordSend webhookURL: "${DISCORD_WEBHOOK}", 
                               title: "Base Manifests Created", 
                               description: "Successfully created and pushed manifests for Apache and CLI images",
                               link: env.BUILD_URL,
                               result: "SUCCESS"
                }
                failure {
                    discordSend webhookURL: "${DISCORD_WEBHOOK}", 
                               title: "Base Manifests Failed", 
                               description: "Failed to create manifests for Apache and CLI images",
                               link: env.BUILD_URL,
                               result: "FAILURE"
                }
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
                        success {
                            discordSend webhookURL: "${DISCORD_WEBHOOK}", 
                                       title: "Dev AMD64 Build Successful", 
                                       description: "Successfully built and pushed ${DOCKER_NAMESPACE}/php${TAG_VERSION}:dev-amd64",
                                       link: env.BUILD_URL,
                                       result: "SUCCESS"
                        }
                        failure {
                            discordSend webhookURL: "${DISCORD_WEBHOOK}", 
                                       title: "Dev AMD64 Build Failed", 
                                       description: "Failed to build ${DOCKER_NAMESPACE}/php${TAG_VERSION}:dev-amd64",
                                       link: env.BUILD_URL,
                                       result: "FAILURE"
                        }
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
                        success {
                            discordSend webhookURL: "${DISCORD_WEBHOOK}", 
                                       title: "Dev ARM64 Build Successful", 
                                       description: "Successfully built and pushed ${DOCKER_NAMESPACE}/php${TAG_VERSION}:dev-arm64",
                                       link: env.BUILD_URL,
                                       result: "SUCCESS"
                        }
                        failure {
                            discordSend webhookURL: "${DISCORD_WEBHOOK}", 
                                       title: "Dev ARM64 Build Failed", 
                                       description: "Failed to build ${DOCKER_NAMESPACE}/php${TAG_VERSION}:dev-arm64",
                                       link: env.BUILD_URL,
                                       result: "FAILURE"
                        }
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
                success {
                    discordSend webhookURL: "${DISCORD_WEBHOOK}", 
                               title: "Dev Manifest Created", 
                               description: "Successfully created and pushed manifest for Dev image",
                               link: env.BUILD_URL,
                               result: "SUCCESS"
                }
                failure {
                    discordSend webhookURL: "${DISCORD_WEBHOOK}", 
                               title: "Dev Manifest Failed", 
                               description: "Failed to create manifest for Dev image",
                               link: env.BUILD_URL,
                               result: "FAILURE"
                }
                always {
                    cleanWs()
                }
            }
        }
    }
    post {
        success {
            node(null) {
                discordSend webhookURL: "${DISCORD_WEBHOOK}", 
                           title: "Build Successful: PHP ${PHP_VERSION} Docker Images", 
                           description: "Successfully built and pushed all PHP ${PHP_VERSION} Docker images:\n" +
                                       "- ${DOCKER_NAMESPACE}/php${TAG_VERSION}:latest (amd64, arm64)\n" +
                                       "- ${DOCKER_NAMESPACE}/php${TAG_VERSION}:cli (amd64, arm64)\n" +
                                       "- ${DOCKER_NAMESPACE}/php${TAG_VERSION}:dev (amd64, arm64)",
                           link: env.BUILD_URL,
                           result: "SUCCESS",
                           footer: "Build #${BUILD_NUMBER} completed in ${currentBuild.durationString}"
            }
        }
        failure {
            node(null) {
                discordSend webhookURL: "${DISCORD_WEBHOOK}", 
                           title: "Build Failed: PHP ${PHP_VERSION} Docker Images", 
                           description: "Failed to build PHP ${PHP_VERSION} Docker images. Check the build logs for details.",
                           link: env.BUILD_URL,
                           result: "FAILURE",
                           footer: "Build #${BUILD_NUMBER} failed after ${currentBuild.durationString}"
            }
        }
        always {
            node(null) {
                // Final cleanup
                sh 'docker buildx prune -f || true'
            }
        }
    }
}
