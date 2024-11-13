pipeline {
    agent any
    environment {
        DOCKER_HUB_CREDENTIALS = credentials('docker-hub-credentials')
        PHP_VERSIONS = ['8.3', '8.2', '8.1']
    }
    stages {
        stage('Test Docker') {
            steps {
                sh 'docker --version'
                sh 'docker ps'
            }
        }
        stage('Set up QEMU and Docker Buildx') {
            steps {
                script {
                    sh 'docker run --rm --privileged multiarch/qemu-user-static --reset -p yes'
                    // Create and use a named builder instance to ensure uniqueness
                    sh 'docker buildx create --name mybuilder --use || docker buildx use mybuilder'
                }
            }
        }
        stage('Build Apache and CLI Images') {
            matrix {
                axes {
                    axis {
                        name 'PHP_VERSION'
                        values PHP_VERSIONS // Use PHP_VERSIONS array from environment
                    }
                    axis {
                        name 'CONFIG'
                        values 'apache', 'cli'
                    }
                }
                stages {
                    stage('Login to Docker Hub') {
                        steps {
                            script {
                                sh 'echo $DOCKER_HUB_CREDENTIALS_PSW | docker login -u $DOCKER_HUB_CREDENTIALS_USR --password-stdin'
                            }
                        }
                    }
                    stage('Checkout Code') {
                        steps {
                            checkout scm
                        }
                    }
                    stage('Set TAG_VERSION') {
                        steps {
                            script {
                                env.TAG_VERSION = PHP_VERSION.replace('.', '')
                            }
                        }
                    }
                    stage('Build and Push Image') {
                        steps {
                            script {
                                def dockerfileSuffix = CONFIG
                                def tagSuffix = dockerfileSuffix == 'apache' ? 'latest' : 'cli'
                                def uniqueTag = "${env.BUILD_NUMBER}-${env.BUILD_ID}" // Unique tag based on Jenkins build details
                                def dockerImage = "ianmgg/php${env.TAG_VERSION}:${tagSuffix}-${uniqueTag}"

                                sh """
                                docker buildx build . \
                                  --platform linux/amd64,linux/arm64 \
                                  --file 8/${PHP_VERSION}/Dockerfile.${dockerfileSuffix} \
                                  --tag ${dockerImage} \
                                  --push
                                """
                            }
                        }
                    }
                }
            }
        }
        stage('Build Dev Images') {
            when {
                expression { currentBuild.result == null || currentBuild.result == 'SUCCESS' }
            }
            matrix {
                axes {
                    axis {
                        name 'PHP_VERSION'
                        values PHP_VERSIONS // Use PHP_VERSIONS array from environment
                    }
                }
                stages {
                    stage('Login to Docker Hub') {
                        steps {
                            script {
                                sh 'echo $DOCKER_HUB_CREDENTIALS_PSW | docker login -u $DOCKER_HUB_CREDENTIALS_USR --password-stdin'
                            }
                        }
                    }
                    stage('Checkout Code') {
                        steps {
                            checkout scm
                        }
                    }
                    stage('Set TAG_VERSION') {
                        steps {
                            script {
                                env.TAG_VERSION = PHP_VERSION.replace('.', '')
                            }
                        }
                    }
                    stage('Build and Push Dev Image') {
                        steps {
                            script {
                                def uniqueTag = "${env.BUILD_NUMBER}-${env.BUILD_ID}" // Unique tag based on Jenkins build details
                                def dockerImage = "ianmgg/php${env.TAG_VERSION}:dev-${uniqueTag}"

                                sh """
                                docker buildx build . \
                                  --platform linux/amd64,linux/arm64 \
                                  --file 8/${PHP_VERSION}/Dockerfile.apache.dev \
                                  --tag ${dockerImage} \
                                  --push
                                """
                            }
                        }
                    }
                    stage('Tag Latest Build') {
                        steps {
                            script {
                                def baseImage = "ianmgg/php${env.TAG_VERSION}"
                                def dockerImage = "${baseImage}:${tagSuffix}-${uniqueTag}"
                                sh """
                                docker tag ${dockerImage} ${baseImage}:${tagSuffix}
                                docker push ${baseImage}:${tagSuffix}
                                """
                            }
                        }
                    }
                }
            }
        }
    }
    post {
        success {
            echo 'All Docker images successfully built and pushed to Docker Hub!'
        }
        failure {
            echo 'Build or push failed.'
        }
    }
}
