pipeline {
    agent any
    environment {
        DOCKER_HUB_CREDENTIALS = credentials('docker-hub-credentials')
    }
    stages {
        stage('Set up QEMU and Docker Buildx') {
            steps {
                script {
                    sh 'docker run --rm --privileged multiarch/qemu-user-static --reset -p yes'
                    sh 'docker buildx create --use || echo "Builder already exists"'
                }
            }
        }
        stage('Build Apache and CLI Images') {
            matrix {
                axes {
                    axis {
                        name 'PHP_VERSION'
                        values '8.3', '8.2', '8.1'
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
                                def dockerImage = "ianmgg/php${env.TAG_VERSION}:${tagSuffix}"

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
                        values '8.3', '8.2', '8.1'
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
                                def dockerImage = "ianmgg/php${env.TAG_VERSION}:dev"

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
