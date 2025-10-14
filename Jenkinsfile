pipeline {
    agent any

    environment {
        GITHUB_CREDENTIALS = 'github-token'
        DOCKERHUB_CREDENTIALS = 'docker-hub-credentials-id'
        BASE_IMAGE = 'matrixdotorg/synapse:latest'
        IMAGE_NAME = 'masarhub/synapse'
        GIT_BRANCH = 'develop' 
    }

    stages {
        stage('Checkout from GitHub') {
            steps {
                git branch: "${GIT_BRANCH}",
                    url: 'https://github.com/masar-edu/chat-backend.git',
                    credentialsId: "${GITHUB_CREDENTIALS}"
            }
        }

        stage('Setup Docker Buildx') {
            steps {
                sh """
                    docker buildx create --use --name multiarch-builder --driver docker-container || true
                    docker buildx inspect --bootstrap
                """
            }
        }

        stage('Login to Docker Hub') {
            steps {
                script {
                    docker.withRegistry('https://index.docker.io/v1/', "${DOCKERHUB_CREDENTIALS}") {
                        echo "Logged in to Docker Hub"
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    // Always rebuild 'latest'
                    def COMMIT_HASH = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    
                    // Pull base image first
                    sh "docker pull ${BASE_IMAGE}"
                    
                    // Build for linux/amd64 platform (single platform for reliability)
                    sh """
                        docker buildx build \
                            --platform linux/amd64 \
                            --no-cache \
                            -t ${IMAGE_NAME}:latest \
                            -t ${IMAGE_NAME}:${COMMIT_HASH} \
                            -f docker/Dockerfile \
                            --push \
                            .
                    """
                }
            }
        }

        // Push handled by buildx in build stage
    }

    post {
        always {
            echo "Cleaning up local Docker images"
            sh "docker rmi ${IMAGE_NAME}:latest || true"
            sh "docker rmi ${IMAGE_NAME}:${COMMIT_HASH} || true"
        }
    }
}
