pipeline {
    agent none // We'll specify agents at the stage level
    options {
        skipDefaultCheckout(false)
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 240, unit: 'MINUTES') // Increased timeout for multiple versions
    }
    environment {
        DOCKER_HUB_CREDENTIALS = credentials('docker-hub-credentials')
        DISCORD_WEBHOOK = credentials('discord-webhook-url')
        DOCKER_BUILDKIT = '1'
        DOCKER_CLI_EXPERIMENTAL = 'enabled'
        DOCKER_NAMESPACE = 'ianmgg'
        // Define PHP versions to build - add or remove versions as needed
        PHP_VERSIONS = """[
            ['version': '8.1', 'tag': '81'],
            ['version': '8.2', 'tag': '82'],
            ['version': '8.3', 'tag': '83'],
            ['version': '8.4', 'tag': '84']
        ]"""
    }
    stages {
        stage('Prepare Workspace') {
            agent { label 'amd64' }
            steps {
                // Send build started notification
                discordSend webhookURL: "${DISCORD_WEBHOOK}", 
                           title: "Build Started: PHP Docker Images", 
                           description: "Building multiple PHP Docker images (Apache, CLI, Dev) for amd64 and arm64 platforms",
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
        
        stage('Build All Versions') {
            steps {
                script {
                    def phpVersions = readJSON text: env.PHP_VERSIONS
                    
                    // Create a map to hold all parallel stages
                    def parallelStages = [:]
                    
                    // For each PHP version, create a set of parallel build stages
                    phpVersions.each { phpConfig ->
                        def phpVersion = phpConfig.version
                        def tagVersion = phpConfig.tag
                        
                        // Create a stage group for this PHP version
                        parallelStages["PHP ${phpVersion}"] = {
                            stage("Build PHP ${phpVersion}") {
                                // Create parallel stages for each image type and architecture
                                def buildStages = [:]
                                
                                // Apache AMD64
                                buildStages["Apache AMD64 (${phpVersion})"] = {
                                    node('amd64') {
                                        try {
                                            unstash 'source-code'
                                            // Login to Docker Hub on this agent
                                            sh 'echo $DOCKER_HUB_CREDENTIALS_PSW | docker login -u $DOCKER_HUB_CREDENTIALS_USR --password-stdin'
                                            
                                            sh """
                                            docker buildx create --name PHPbuilder-apache-amd64-${phpVersion}-${BUILD_NUMBER} --use || true
                                            docker buildx inspect PHPbuilder-apache-amd64-${phpVersion}-${BUILD_NUMBER} --bootstrap
                                            
                                            docker buildx build . \\
                                              --progress=plain \\
                                              --platform linux/amd64 \\
                                              --file 8/${phpVersion}/Dockerfile.apache \\
                                              --tag ${DOCKER_NAMESPACE}/php${tagVersion}:latest-amd64 \\
                                              --push \\
                                              --no-cache \\
                                              --build-arg BUILDKIT_INLINE_CACHE=1
                                            """
                                            
                                            discordSend webhookURL: "${DISCORD_WEBHOOK}", 
                                                       title: "Apache AMD64 (${phpVersion}) Build Successful", 
                                                       description: "Successfully built and pushed ${DOCKER_NAMESPACE}/php${tagVersion}:latest-amd64",
                                                       link: env.BUILD_URL,
                                                       result: "SUCCESS"
                                        } catch (Exception e) {
                                            discordSend webhookURL: "${DISCORD_WEBHOOK}", 
                                                       title: "Apache AMD64 (${phpVersion}) Build Failed", 
                                                       description: "Failed to build ${DOCKER_NAMESPACE}/php${tagVersion}:latest-amd64",
                                                       link: env.BUILD_URL,
                                                       result: "FAILURE"
                                            throw e
                                        } finally {
                                            sh "docker buildx rm PHPbuilder-apache-amd64-${phpVersion}-${BUILD_NUMBER} || true"
                                            cleanWs()
                                        }
                                    }
                                }
                                
                                // Apache ARM64
                                buildStages["Apache ARM64 (${phpVersion})"] = {
                                    node('arm64') {
                                        try {
                                            unstash 'source-code'
                                            // Login to Docker Hub on this agent
                                            sh 'echo $DOCKER_HUB_CREDENTIALS_PSW | docker login -u $DOCKER_HUB_CREDENTIALS_USR --password-stdin'
                                            
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
                                            
                                            sh """
                                            docker buildx create --name PHPbuilder-apache-arm64-${phpVersion}-${BUILD_NUMBER} --use || true
                                            docker buildx inspect PHPbuilder-apache-arm64-${phpVersion}-${BUILD_NUMBER} --bootstrap
                                            
                                            docker buildx build . \\
                                              --progress=plain \\
                                              --platform linux/arm64 \\
                                              --file 8/${phpVersion}/Dockerfile.apache \\
                                              --tag ${DOCKER_NAMESPACE}/php${tagVersion}:latest-arm64 \\
                                              --push \\
                                              --no-cache \\
                                              --build-arg BUILDKIT_INLINE_CACHE=1 \\
                                              --memory=12g \\
                                              --memory-swap=24g
                                            """
                                            
                                            discordSend webhookURL: "${DISCORD_WEBHOOK}", 
                                                       title: "Apache ARM64 (${phpVersion}) Build Successful", 
                                                       description: "Successfully built and pushed ${DOCKER_NAMESPACE}/php${tagVersion}:latest-arm64",
                                                       link: env.BUILD_URL,
                                                       result: "SUCCESS"
                                        } catch (Exception e) {
                                            discordSend webhookURL: "${DISCORD_WEBHOOK}", 
                                                       title: "Apache ARM64 (${phpVersion}) Build Failed", 
                                                       description: "Failed to build ${DOCKER_NAMESPACE}/php${tagVersion}:latest-arm64",
                                                       link: env.BUILD_URL,
                                                       result: "FAILURE"
                                            throw e
                                        } finally {
                                            sh "docker buildx rm PHPbuilder-apache-arm64-${phpVersion}-${BUILD_NUMBER} || true"
                                            cleanWs()
                                        }
                                    }
                                }
                                
                                // CLI AMD64
                                buildStages["CLI AMD64 (${phpVersion})"] = {
                                    node('amd64') {
                                        try {
                                            unstash 'source-code'
                                            // Login to Docker Hub on this agent
                                            sh 'echo $DOCKER_HUB_CREDENTIALS_PSW | docker login -u $DOCKER_HUB_CREDENTIALS_USR --password-stdin'
                                            
                                            sh """
                                            docker buildx create --name PHPbuilder-cli-amd64-${phpVersion}-${BUILD_NUMBER} --use || true
                                            docker buildx inspect PHPbuilder-cli-amd64-${phpVersion}-${BUILD_NUMBER} --bootstrap
                                            
                                            docker buildx build . \\
                                              --progress=plain \\
                                              --platform linux/amd64 \\
                                              --file 8/${phpVersion}/Dockerfile.cli \\
                                              --tag ${DOCKER_NAMESPACE}/php${tagVersion}:cli-amd64 \\
                                              --push \\
                                              --no-cache \\
                                              --build-arg BUILDKIT_INLINE_CACHE=1
                                            """
                                            
                                            discordSend webhookURL: "${DISCORD_WEBHOOK}", 
                                                       title: "CLI AMD64 (${phpVersion}) Build Successful", 
                                                       description: "Successfully built and pushed ${DOCKER_NAMESPACE}/php${tagVersion}:cli-amd64",
                                                       link: env.BUILD_URL,
                                                       result: "SUCCESS"
                                        } catch (Exception e) {
                                            discordSend webhookURL: "${DISCORD_WEBHOOK}", 
                                                       title: "CLI AMD64 (${phpVersion}) Build Failed", 
                                                       description: "Failed to build ${DOCKER_NAMESPACE}/php${tagVersion}:cli-amd64",
                                                       link: env.BUILD_URL,
                                                       result: "FAILURE"
                                            throw e
                                        } finally {
                                            sh "docker buildx rm PHPbuilder-cli-amd64-${phpVersion}-${BUILD_NUMBER} || true"
                                            cleanWs()
                                        }
                                    }
                                }
                                
                                // CLI ARM64
                                buildStages["CLI ARM64 (${phpVersion})"] = {
                                    node('arm64') {
                                        try {
                                            unstash 'source-code'
                                            // Login to Docker Hub on this agent
                                            sh 'echo $DOCKER_HUB_CREDENTIALS_PSW | docker login -u $DOCKER_HUB_CREDENTIALS_USR --password-stdin'
                                            
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
                                            
                                            sh """
                                            docker buildx create --name PHPbuilder-cli-arm64-${phpVersion}-${BUILD_NUMBER} --use || true
                                            docker buildx inspect PHPbuilder-cli-arm64-${phpVersion}-${BUILD_NUMBER} --bootstrap
                                            
                                            docker buildx build . \\
                                              --progress=plain \\
                                              --platform linux/arm64 \\
                                              --file 8/${phpVersion}/Dockerfile.cli \\
                                              --tag ${DOCKER_NAMESPACE}/php${tagVersion}:cli-arm64 \\
                                              --push \\
                                              --no-cache \\
                                              --build-arg BUILDKIT_INLINE_CACHE=1 \\
                                              --memory=12g \\
                                              --memory-swap=24g
                                            """
                                            
                                            discordSend webhookURL: "${DISCORD_WEBHOOK}", 
                                                       title: "CLI ARM64 (${phpVersion}) Build Successful", 
                                                       description: "Successfully built and pushed ${DOCKER_NAMESPACE}/php${tagVersion}:cli-arm64",
                                                       link: env.BUILD_URL,
                                                       result: "SUCCESS"
                                        } catch (Exception e) {
                                            discordSend webhookURL: "${DISCORD_WEBHOOK}", 
                                                       title: "CLI ARM64 (${phpVersion}) Build Failed", 
                                                       description: "Failed to build ${DOCKER_NAMESPACE}/php${tagVersion}:cli-arm64",
                                                       link: env.BUILD_URL,
                                                       result: "FAILURE"
                                            throw e
                                        } finally {
                                            sh "docker buildx rm PHPbuilder-cli-arm64-${phpVersion}-${BUILD_NUMBER} || true"
                                            cleanWs()
                                        }
                                    }
                                }
                                
                                // Run all builds for this PHP version in parallel
                                parallel buildStages
                                
                                // Create base manifests for this PHP version
                                stage("Create Base Manifests (${phpVersion})") {
                                    node('amd64') {
                                        try {
                                            unstash 'source-code'
                                            // Login to Docker Hub on this agent
                                            sh 'echo $DOCKER_HUB_CREDENTIALS_PSW | docker login -u $DOCKER_HUB_CREDENTIALS_USR --password-stdin'
                                            
                                            sh """
                                            # Create Apache manifest
                                            docker buildx imagetools create --tag ${DOCKER_NAMESPACE}/php${tagVersion}:latest \\
                                              ${DOCKER_NAMESPACE}/php${tagVersion}:latest-amd64 \\
                                              ${DOCKER_NAMESPACE}/php${tagVersion}:latest-arm64
                                            
                                            # Create CLI manifest
                                            docker buildx imagetools create --tag ${DOCKER_NAMESPACE}/php${tagVersion}:cli \\
                                              ${DOCKER_NAMESPACE}/php${tagVersion}:cli-amd64 \\
                                              ${DOCKER_NAMESPACE}/php${tagVersion}:cli-arm64
                                            
                                            # Verify the manifests
                                            docker buildx imagetools inspect ${DOCKER_NAMESPACE}/php${tagVersion}:latest
                                            docker buildx imagetools inspect ${DOCKER_NAMESPACE}/php${tagVersion}:cli
                                            """
                                            
                                            discordSend webhookURL: "${DISCORD_WEBHOOK}", 
                                                       title: "Base Manifests (${phpVersion}) Created", 
                                                       description: "Successfully created and pushed manifests for Apache and CLI images",
                                                       link: env.BUILD_URL,
                                                       result: "SUCCESS"
                                        } catch (Exception e) {
                                            discordSend webhookURL: "${DISCORD_WEBHOOK}", 
                                                       title: "Base Manifests (${phpVersion}) Failed", 
                                                       description: "Failed to create manifests for Apache and CLI images",
                                                       link: env.BUILD_URL,
                                                       result: "FAILURE"
                                            throw e
                                        } finally {
                                            cleanWs()
                                        }
                                    }
                                }
                                
                                // Build Dev images for this PHP version
                                def devBuildStages = [:]
                                
                                // Dev AMD64
                                devBuildStages["Dev AMD64 (${phpVersion})"] = {
                                    node('amd64') {
                                        try {
                                            unstash 'source-code'
                                            // Login to Docker Hub on this agent
                                            sh 'echo $DOCKER_HUB_CREDENTIALS_PSW | docker login -u $DOCKER_HUB_CREDENTIALS_USR --password-stdin'
                                            
                                            sh """
                                            docker buildx create --name PHPbuilder-dev-amd64-${phpVersion}-${BUILD_NUMBER} --use || true
                                            docker buildx inspect PHPbuilder-dev-amd64-${phpVersion}-${BUILD_NUMBER} --bootstrap
                                            
                                            docker buildx build . \\
                                              --progress=plain \\
                                              --platform linux/amd64 \\
                                              --file 8/${phpVersion}/Dockerfile.apache.dev \\
                                              --tag ${DOCKER_NAMESPACE}/php${tagVersion}:dev-amd64 \\
                                              --push \\
                                              --no-cache \\
                                              --build-arg BUILDKIT_INLINE_CACHE=1
                                            """
                                            
                                            discordSend webhookURL: "${DISCORD_WEBHOOK}", 
                                                       title: "Dev AMD64 (${phpVersion}) Build Successful", 
                                                       description: "Successfully built and pushed ${DOCKER_NAMESPACE}/php${tagVersion}:dev-amd64",
                                                       link: env.BUILD_URL,
                                                       result: "SUCCESS"
                                        } catch (Exception e) {
                                            discordSend webhookURL: "${DISCORD_WEBHOOK}", 
                                                       title: "Dev AMD64 (${phpVersion}) Build Failed", 
                                                       description: "Failed to build ${DOCKER_NAMESPACE}/php${tagVersion}:dev-amd64",
                                                       link: env.BUILD_URL,
                                                       result: "FAILURE"
                                            throw e
                                        } finally {
                                            sh "docker buildx rm PHPbuilder-dev-amd64-${phpVersion}-${BUILD_NUMBER} || true"
                                            cleanWs()
                                        }
                                    }
                                }
                                
                                // Dev ARM64
                                devBuildStages["Dev ARM64 (${phpVersion})"] = {
                                    node('arm64') {
                                        try {
                                            unstash 'source-code'
                                            // Login to Docker Hub on this agent
                                            sh 'echo $DOCKER_HUB_CREDENTIALS_PSW | docker login -u $DOCKER_HUB_CREDENTIALS_USR --password-stdin'
                                            
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
                                            
                                            sh """
                                            docker buildx create --name PHPbuilder-dev-arm64-${phpVersion}-${BUILD_NUMBER} --use || true
                                            docker buildx inspect PHPbuilder-dev-arm64-${phpVersion}-${BUILD_NUMBER} --bootstrap
                                            
                                            docker buildx build . \\
                                              --progress=plain \\
                                              --platform linux/arm64 \\
                                              --file 8/${phpVersion}/Dockerfile.apache.dev \\
                                              --tag ${DOCKER_NAMESPACE}/php${tagVersion}:dev-arm64 \\
                                              --push \\
                                              --no-cache \\
                                              --build-arg BUILDKIT_INLINE_CACHE=1 \\
                                              --memory=12g \\
                                              --memory-swap=24g
                                            """
                                            
                                            discordSend webhookURL: "${DISCORD_WEBHOOK}", 
                                                       title: "Dev ARM64 (${phpVersion}) Build Successful", 
                                                       description: "Successfully built and pushed ${DOCKER_NAMESPACE}/php${tagVersion}:dev-arm64",
                                                       link: env.BUILD_URL,
                                                       result: "SUCCESS"
                                        } catch (Exception e) {
                                            discordSend webhookURL: "${DISCORD_WEBHOOK}", 
                                                       title: "Dev ARM64 (${phpVersion}) Build Failed", 
                                                       description: "Failed to build ${DOCKER_NAMESPACE}/php${tagVersion}:dev-arm64",
                                                       link: env.BUILD_URL,
                                                       result: "FAILURE"
                                            throw e
                                        } finally {
                                            sh "docker buildx rm PHPbuilder-dev-arm64-${phpVersion}-${BUILD_NUMBER} || true"
                                            cleanWs()
                                        }
                                    }
                                }
                                
                                // Run Dev builds in parallel
                                parallel devBuildStages
                                
                                // Create Dev manifest for this PHP version
                                stage("Create Dev Manifest (${phpVersion})") {
                                    node('amd64') {
                                        try {
                                            unstash 'source-code'
                                            // Login to Docker Hub on this agent
                                            sh 'echo $DOCKER_HUB_CREDENTIALS_PSW | docker login -u $DOCKER_HUB_CREDENTIALS_USR --password-stdin'
                                            
                                            sh """
                                            # Create Dev manifest
                                            docker buildx imagetools create --tag ${DOCKER_NAMESPACE}/php${tagVersion}:dev \\
                                              ${DOCKER_NAMESPACE}/php${tagVersion}:dev-amd64 \\
                                              ${DOCKER_NAMESPACE}/php${tagVersion}:dev-arm64
                                            
                                            # Verify the manifest
                                            docker buildx imagetools inspect ${DOCKER_NAMESPACE}/php${tagVersion}:dev
                                            """
                                            
                                            discordSend webhookURL: "${DISCORD_WEBHOOK}", 
                                                       title: "Dev Manifest (${phpVersion}) Created", 
                                                       description: "Successfully created and pushed manifest for Dev image",
                                                       link: env.BUILD_URL,
                                                       result: "SUCCESS"
                                        } catch (Exception e) {
                                            discordSend webhookURL: "${DISCORD_WEBHOOK}", 
                                                       title: "Dev Manifest (${phpVersion}) Failed", 
                                                       description: "Failed to create manifest for Dev image",
                                                       link: env.BUILD_URL,
                                                       result: "FAILURE"
                                            throw e
                                        } finally {
                                            cleanWs()
                                        }
                                    }
                                }
                            }
                        }
                    }
                    
                    // Run all PHP version builds in parallel
                    parallel parallelStages
                }
            }
        }
    }
    post {
        success {
            node(null) {
                script {
                    def phpVersions = readJSON text: env.PHP_VERSIONS
                    def versionsList = phpVersions.collect { "PHP ${it.version}" }.join(", ")
                    def imagesList = phpVersions.collect { phpConfig ->
                        def tagVersion = phpConfig.tag
                        return "- ${DOCKER_NAMESPACE}/php${tagVersion}:latest (amd64, arm64)\n" +
                               "- ${DOCKER_NAMESPACE}/php${tagVersion}:cli (amd64, arm64)\n" +
                               "- ${DOCKER_NAMESPACE}/php${tagVersion}:dev (amd64, arm64)"
                    }.join("\n")
                    
                    discordSend webhookURL: "${DISCORD_WEBHOOK}", 
                               title: "Build Successful: PHP Docker Images", 
                               description: "Successfully built and pushed all PHP Docker images for ${versionsList}:\n${imagesList}",
                               link: env.BUILD_URL,
                               result: "SUCCESS",
                               footer: "Build #${BUILD_NUMBER} completed in ${currentBuild.durationString}"
                }
            }
        }
        failure {
            node(null) {
                script {
                    def phpVersions = readJSON text: env.PHP_VERSIONS
                    def versionsList = phpVersions.collect { "PHP ${it.version}" }.join(", ")
                    
                    discordSend webhookURL: "${DISCORD_WEBHOOK}", 
                               title: "Build Failed: PHP Docker Images", 
                               description: "Failed to build PHP Docker images for ${versionsList}. Check the build logs for details.",
                               link: env.BUILD_URL,
                               result: "FAILURE",
                               footer: "Build #${BUILD_NUMBER} failed after ${currentBuild.durationString}"
                }
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
