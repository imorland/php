pipeline {
    agent any
    environment {
        DOCKER_HUB_CREDENTIALS = credentials('docker-hub-credentials')
        DOCKER_BUILDKIT = "1"
        DOCKER_CLI_EXPERIMENTAL = "enabled"
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
        stage('Build Apache, CLI and Dev Images') {
            steps {
                script {
                    def phpVersions = ['8.3']

                    for (phpVersion in phpVersions) {
                        def tagVersion = phpVersion.replace('.', '')
                        
                        // Login to Docker Hub
                        sh 'echo $DOCKER_HUB_CREDENTIALS_PSW | docker login -u $DOCKER_HUB_CREDENTIALS_USR --password-stdin'
                        
                        // Build Apache image
                        def apacheImage = "ianmgg/php${tagVersion}:latest"
                        sh """
                        docker buildx build . \
                          --platform linux/amd64,linux/arm64 \
                          --file 8/${phpVersion}/Dockerfile.apache \
                          --tag ${apacheImage} \
                          --push
                        """

                        // Build CLI image
                        def cliImage = "ianmgg/php${tagVersion}:cli"
                        sh """
                        docker buildx build . \
                          --platform linux/amd64,linux/arm64 \
                          --file 8/${phpVersion}/Dockerfile.cli \
                          --tag ${cliImage} \
                          --push
                        """

                        // Build Dev image
                        def devImage = "ianmgg/php${tagVersion}:dev"
                        sh """
                        docker buildx build . \
                          --platform linux/amd64,linux/arm64 \
                          --file 8/${phpVersion}/Dockerfile.apache.dev \
                          --tag ${devImage} \
                          --push
                        """
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
