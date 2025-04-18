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
            agent { label 'amd64' }
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
                    agent { label 'amd64' }
                    steps {
                        unstash 'source-code'
                        script {
                            // Login to Docker Hub on this agent
                            sh 'echo $DOCKER_HUB_CREDENTIALS_PSW | docker login -u $DOCKER_HUB_CREDENTIALS_USR --password-stdin'
                            
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
                    agent { label 'arm64' }
                    steps {
                        unstash 'source-code'
                        script {
                            // Login to Docker Hub on this agent
                            sh 'echo $DOCKER_HUB_CREDENTIALS_PSW | docker login -u $DOCKER_HUB_CREDENTIALS_USR --password-stdin'
                            
                            // Check if we're on a native ARM64 agent
                            sh '''
                            HOST_ARCH=$(uname -m)
                            if [ "$HOST_ARCH" != "aarch64" ]; then
                              # Only set up QEMU if we're not on a native ARM64 host
                              echo "Setting up QEMU for ARM64 emulation on $HOST_ARCH..."
                              docker run --rm --privileged tonistiigi/binfmt:latest --install arm64 || docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
                              echo "10" > /proc/sys/vm/nr_hugepages || true  # Optimize memory for QEMU
                            else
                              echo "Running on native ARM64 host, no QEMU needed for ARM64 builds"
                            fi
                            '''
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
                              --build-arg BUILDKIT_INLINE_CACHE=1 \
                              --memory=12g \
                              --memory-swap=24g
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
                    agent { label 'amd64' }
                    steps {
                        unstash 'source-code'
                        script {
                            // Login to Docker Hub on this agent
                            sh 'echo $DOCKER_HUB_CREDENTIALS_PSW | docker login -u $DOCKER_HUB_CREDENTIALS_USR --password-stdin'
                            
                            // Check if we're on a native ARM64 agent that needs QEMU for AMD64 builds
                            sh '''
                            HOST_ARCH=$(uname -m)
                            if [ "$HOST_ARCH" = "aarch64" ]; then
                              # Set up QEMU for AMD64 emulation on ARM64 host
                              echo "Setting up QEMU for AMD64 emulation on ARM64 host..."
                              docker run --rm --privileged --platform linux/arm64 tonistiigi/binfmt:latest --install amd64 || true
                            else
                              echo "Running on x86_64 host, no QEMU needed for AMD64 builds"
                            fi
                            '''
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
                    agent { label 'arm64' }
                    steps {
                        unstash 'source-code'
                        script {
                            // Login to Docker Hub on this agent
                            sh 'echo $DOCKER_HUB_CREDENTIALS_PSW | docker login -u $DOCKER_HUB_CREDENTIALS_USR --password-stdin'
                            
                            // Check if we're on a native ARM64 agent that needs QEMU for AMD64 builds
                            sh '''
                            HOST_ARCH=$(uname -m)
                            if [ "$HOST_ARCH" = "aarch64" ]; then
                              # Set up QEMU for AMD64 emulation on ARM64 host
                              echo "Setting up QEMU for AMD64 emulation on ARM64 host..."
                              docker run --rm --privileged --platform linux/arm64 tonistiigi/binfmt:latest --install amd64 || true
                            else
                              echo "Running on x86_64 host, no QEMU needed for AMD64 builds"
                            fi
                            '''
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
                              --build-arg BUILDKIT_INLINE_CACHE=1 \
                              --memory=12g \
                              --memory-swap=24g
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
            agent { label 'amd64' }
            steps {
                unstash 'source-code'
                script {
                    // Login to Docker Hub on this agent
                    sh 'echo $DOCKER_HUB_CREDENTIALS_PSW | docker login -u $DOCKER_HUB_CREDENTIALS_USR --password-stdin'
                    
                    // Check if the architecture-specific images are manifest lists
                    sh '''
                    # Remove existing manifests
                    docker manifest rm ${DOCKER_NAMESPACE}/php${TAG_VERSION}:latest 2>/dev/null || true
                    docker manifest rm ${DOCKER_NAMESPACE}/php${TAG_VERSION}:cli 2>/dev/null || true
                    
                    # Function to check if an image is a manifest list
                    check_image() {
                      local image=$1
                      local is_manifest=false
                      
                      # Check if the image exists and is a manifest list
                      if docker manifest inspect $image &>/dev/null; then
                        if docker manifest inspect $image | grep -q "manifests"; then
                          echo "$image is a manifest list"
                          is_manifest=true
                        else
                          echo "$image is a single-architecture image"
                        fi
                      else
                        echo "$image does not exist"
                      fi
                      
                      echo $is_manifest
                    }
                    
                    # Check all images
                    AMD64_APACHE_IS_MANIFEST=$(check_image ${DOCKER_NAMESPACE}/php${TAG_VERSION}:latest-amd64)
                    ARM64_APACHE_IS_MANIFEST=$(check_image ${DOCKER_NAMESPACE}/php${TAG_VERSION}:latest-arm64)
                    AMD64_CLI_IS_MANIFEST=$(check_image ${DOCKER_NAMESPACE}/php${TAG_VERSION}:cli-amd64)
                    ARM64_CLI_IS_MANIFEST=$(check_image ${DOCKER_NAMESPACE}/php${TAG_VERSION}:cli-arm64)
                    
                    # Save results to files for later use
                    echo $AMD64_APACHE_IS_MANIFEST > amd64_apache_manifest.txt
                    echo $ARM64_APACHE_IS_MANIFEST > arm64_apache_manifest.txt
                    echo $AMD64_CLI_IS_MANIFEST > amd64_cli_manifest.txt
                    echo $ARM64_CLI_IS_MANIFEST > arm64_cli_manifest.txt
                    '''
                    
                    // Create Apache manifest using digest references
                    sh '''
                    # Remove any existing manifests
                    docker manifest rm ${DOCKER_NAMESPACE}/php${TAG_VERSION}:latest 2>/dev/null || true
                    
                    # Extract digests from the manifest lists
                    echo "Extracting digests from manifest lists..."
                    
                    # For AMD64 Apache
                    AMD64_APACHE_DIGEST=$(docker manifest inspect ${DOCKER_NAMESPACE}/php${TAG_VERSION}:latest-amd64 | grep -A 10 '"architecture": "amd64"' | grep digest | head -1 | awk -F\\" '{print $4}')
                    echo "AMD64 Apache digest: $AMD64_APACHE_DIGEST"
                    
                    # For ARM64 Apache
                    ARM64_APACHE_DIGEST=$(docker manifest inspect ${DOCKER_NAMESPACE}/php${TAG_VERSION}:latest-arm64 | grep -A 10 '"architecture": "arm64"' | grep digest | head -1 | awk -F\\" '{print $4}')
                    echo "ARM64 Apache digest: $ARM64_APACHE_DIGEST"
                    
                    # Create the manifest with digest references
                    echo "Creating Apache manifest with both architectures..."
                    docker manifest create ${DOCKER_NAMESPACE}/php${TAG_VERSION}:latest \
                      --amend ${DOCKER_NAMESPACE}/php${TAG_VERSION}@${AMD64_APACHE_DIGEST} \
                      --amend ${DOCKER_NAMESPACE}/php${TAG_VERSION}@${ARM64_APACHE_DIGEST}
                    
                    # Push the manifest
                    echo "Pushing Apache manifest..."
                    docker manifest push --purge ${DOCKER_NAMESPACE}/php${TAG_VERSION}:latest
                    
                    # Verify the manifest includes both architectures
                    echo "Verifying Apache manifest..."
                    docker manifest inspect ${DOCKER_NAMESPACE}/php${TAG_VERSION}:latest
                    '''
                    
                    // Create CLI manifest using digest references
                    sh '''
                    # Remove any existing manifests
                    docker manifest rm ${DOCKER_NAMESPACE}/php${TAG_VERSION}:cli 2>/dev/null || true
                    
                    # Extract digests from the manifest lists
                    echo "Extracting digests from manifest lists..."
                    
                    # For AMD64 CLI
                    AMD64_CLI_DIGEST=$(docker manifest inspect ${DOCKER_NAMESPACE}/php${TAG_VERSION}:cli-amd64 | grep -A 10 '"architecture": "amd64"' | grep digest | head -1 | awk -F\\" '{print $4}')
                    echo "AMD64 CLI digest: $AMD64_CLI_DIGEST"
                    
                    # For ARM64 CLI
                    ARM64_CLI_DIGEST=$(docker manifest inspect ${DOCKER_NAMESPACE}/php${TAG_VERSION}:cli-arm64 | grep -A 10 '"architecture": "arm64"' | grep digest | head -1 | awk -F\\" '{print $4}')
                    echo "ARM64 CLI digest: $ARM64_CLI_DIGEST"
                    
                    # Create the manifest with digest references
                    echo "Creating CLI manifest with both architectures..."
                    docker manifest create ${DOCKER_NAMESPACE}/php${TAG_VERSION}:cli \
                      --amend ${DOCKER_NAMESPACE}/php${TAG_VERSION}@${AMD64_CLI_DIGEST} \
                      --amend ${DOCKER_NAMESPACE}/php${TAG_VERSION}@${ARM64_CLI_DIGEST}
                    
                    # Push the manifest
                    echo "Pushing CLI manifest..."
                    docker manifest push --purge ${DOCKER_NAMESPACE}/php${TAG_VERSION}:cli
                    
                    # Verify the manifest includes both architectures
                    echo "Verifying CLI manifest..."
                    docker manifest inspect ${DOCKER_NAMESPACE}/php${TAG_VERSION}:cli
                    '''
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
                    agent { label 'amd64' }
                    steps {
                        unstash 'source-code'
                        script {
                            // Check if we're on a native ARM64 agent that needs QEMU for AMD64 builds
                            sh '''
                            HOST_ARCH=$(uname -m)
                            if [ "$HOST_ARCH" = "aarch64" ]; then
                              # Set up QEMU for AMD64 emulation on ARM64 host
                              echo "Setting up QEMU for AMD64 emulation on ARM64 host..."
                              docker run --rm --privileged --platform linux/arm64 tonistiigi/binfmt:latest --install amd64 || true
                            else
                              echo "Running on x86_64 host, no QEMU needed for AMD64 builds"
                            fi
                            '''
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
                    agent { label 'arm64' }
                    steps {
                        unstash 'source-code'
                        script {
                            // Login to Docker Hub on this agent
                            sh 'echo $DOCKER_HUB_CREDENTIALS_PSW | docker login -u $DOCKER_HUB_CREDENTIALS_USR --password-stdin'
                            
                            // Check if we're on a native ARM64 agent
                            sh '''
                            HOST_ARCH=$(uname -m)
                            if [ "$HOST_ARCH" != "aarch64" ]; then
                              # Only set up QEMU if we're not on a native ARM64 host
                              echo "Setting up QEMU for ARM64 emulation on $HOST_ARCH..."
                              docker run --rm --privileged tonistiigi/binfmt:latest --install arm64 || docker run --rm --privileged multiarch/qemu-user-static --reset -p yes
                              echo "10" > /proc/sys/vm/nr_hugepages || true  # Optimize memory for QEMU
                            else
                              echo "Running on native ARM64 host, no QEMU needed for ARM64 builds"
                            fi
                            '''
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
                              --build-arg BUILDKIT_INLINE_CACHE=1 \
                              --memory=12g \
                              --memory-swap=24g
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
            agent { label 'amd64' }
            steps {
                unstash 'source-code'
                script {
                    // Login to Docker Hub on this agent
                    sh 'echo $DOCKER_HUB_CREDENTIALS_PSW | docker login -u $DOCKER_HUB_CREDENTIALS_USR --password-stdin'
                    
                    // Check if the architecture-specific images are manifest lists
                    sh '''
                    # Remove existing manifest
                    docker manifest rm ${DOCKER_NAMESPACE}/php${TAG_VERSION}:dev 2>/dev/null || true
                    
                    # Function to check if an image is a manifest list
                    check_image() {
                      local image=$1
                      local is_manifest=false
                      
                      # Check if the image exists and is a manifest list
                      if docker manifest inspect $image &>/dev/null; then
                        if docker manifest inspect $image | grep -q "manifests"; then
                          echo "$image is a manifest list"
                          is_manifest=true
                        else
                          echo "$image is a single-architecture image"
                        fi
                      else
                        echo "$image does not exist"
                      fi
                      
                      echo $is_manifest
                    }
                    
                    # Check dev images
                    AMD64_DEV_IS_MANIFEST=$(check_image ${DOCKER_NAMESPACE}/php${TAG_VERSION}:dev-amd64)
                    ARM64_DEV_IS_MANIFEST=$(check_image ${DOCKER_NAMESPACE}/php${TAG_VERSION}:dev-arm64)
                    
                    # Save results to files for later use
                    echo $AMD64_DEV_IS_MANIFEST > amd64_dev_manifest.txt
                    echo $ARM64_DEV_IS_MANIFEST > arm64_dev_manifest.txt
                    '''
                    
                    // Create Dev manifest using digest references
                    sh '''
                    # Remove any existing manifests
                    docker manifest rm ${DOCKER_NAMESPACE}/php${TAG_VERSION}:dev 2>/dev/null || true
                    
                    # Extract digests from the manifest lists
                    echo "Extracting digests from manifest lists..."
                    
                    # For AMD64 Dev
                    AMD64_DEV_DIGEST=$(docker manifest inspect ${DOCKER_NAMESPACE}/php${TAG_VERSION}:dev-amd64 | grep -A 10 '"architecture": "amd64"' | grep digest | head -1 | awk -F\\" '{print $4}')
                    echo "AMD64 Dev digest: $AMD64_DEV_DIGEST"
                    
                    # For ARM64 Dev
                    ARM64_DEV_DIGEST=$(docker manifest inspect ${DOCKER_NAMESPACE}/php${TAG_VERSION}:dev-arm64 | grep -A 10 '"architecture": "arm64"' | grep digest | head -1 | awk -F\\" '{print $4}')
                    echo "ARM64 Dev digest: $ARM64_DEV_DIGEST"
                    
                    # Create the manifest with digest references
                    echo "Creating Dev manifest with both architectures..."
                    docker manifest create ${DOCKER_NAMESPACE}/php${TAG_VERSION}:dev \
                      --amend ${DOCKER_NAMESPACE}/php${TAG_VERSION}@${AMD64_DEV_DIGEST} \
                      --amend ${DOCKER_NAMESPACE}/php${TAG_VERSION}@${ARM64_DEV_DIGEST}
                    
                    # Push the manifest
                    echo "Pushing Dev manifest..."
                    docker manifest push --purge ${DOCKER_NAMESPACE}/php${TAG_VERSION}:dev
                    
                    # Verify the manifest includes both architectures
                    echo "Verifying Dev manifest..."
                    docker manifest inspect ${DOCKER_NAMESPACE}/php${TAG_VERSION}:dev
                    '''
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
