pipeline {
    agent any

    environment {
        GITHUB_CREDENTIALS = 'github-token'
        DOCKERHUB_CREDENTIALS = 'docker-hub-credentials-id'
        BASE_IMAGE = 'matrixdotorg/synapse:latest'
        IMAGE_NAME = 'masarhub/synapse'
        GIT_BRANCH = 'develop'
        COMMIT_HASH = ''
        SKIP_DOCKER_LOGIN = 'false' // Set to 'true' to skip Docker login for testing
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

        stage('Build and Push Docker Image') {
            steps {
                script {
                    // Get commit hash and set as environment variable
                    env.COMMIT_HASH = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    
                    // Login to Docker Hub
                    withCredentials([usernamePassword(credentialsId: "${DOCKERHUB_CREDENTIALS}", 
                                                    passwordVariable: 'DOCKER_PASSWORD', 
                                                    usernameVariable: 'DOCKER_USERNAME')]) {
                        
                        echo "Attempting to login to Docker Hub with username: ${DOCKER_USERNAME}"
                        
                        def loginResult = sh(
                            script: 'echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin',
                            returnStatus: true
                        )
                        
                        if (loginResult != 0) {
                            error "Docker Hub login failed. Please check credentials in Jenkins."
                        }
                        
                        echo "Successfully logged in to Docker Hub"
                        
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
                        
                        echo "Successfully built and pushed images:"
                        echo "- ${IMAGE_NAME}:latest"
                        echo "- ${IMAGE_NAME}:${env.COMMIT_HASH}"
                    }
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
