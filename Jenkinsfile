pipeline {
    agent any

    environment {
        GITHUB_CREDENTIALS = 'github-token'
        DOCKERHUB_CREDENTIALS = 'docker-hub-credentials-id'
        BASE_IMAGE = 'matrixdotorg/synapse:latest'
        IMAGE_NAME = 'masarhub/synapse'
        GIT_BRANCH = 'develop'
        COMMIT_HASH = ''
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
                    # Remove existing builder if it exists
                    docker buildx rm multiarch-builder || true
                    
                    # Create fresh builder
                    docker buildx create --use --name multiarch-builder --driver docker-container
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
                    // Get commit hash and set as environment variable
                    env.COMMIT_HASH = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    
                    // Pull base image first
                    sh "docker pull ${BASE_IMAGE}"
                    
                    // Build for linux/amd64 platform (single platform for reliability)
                    sh """
                        docker buildx build \
                            --platform linux/amd64 \
                            --no-cache \
                            -t ${IMAGE_NAME}:latest \
                            -t ${IMAGE_NAME}:${env.COMMIT_HASH} \
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
            script {
                echo "Cleaning up Docker buildx and local resources"
                
                // Clean up buildx cache (keep builder for reuse)
                sh "docker buildx prune -f || true"
                
                // Clean up any dangling images
                sh "docker image prune -f || true"
                
                // Keep the builder for next runs (don't remove)
                echo "Keeping multiarch-builder for future builds"
                
                echo "Pipeline completed. Images pushed to Docker Hub:"
                echo "- ${IMAGE_NAME}:latest"
                if (env.COMMIT_HASH) {
                    echo "- ${IMAGE_NAME}:${env.COMMIT_HASH}"
                }
            }
        }
        
        success {
            echo "✅ Build and push successful!"
        }
        
        failure {
            echo "❌ Build failed. Check the logs above."
        }
    }
}
