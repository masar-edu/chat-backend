pipeline {
    agent any

    environment {
        GITHUB_CREDENTIALS = 'github-token'
        DOCKERHUB_CREDENTIALS = 'dockerhub-credentials'
        BASE_IMAGE = 'matrixdotorg/synapse:latest'
        IMAGE_NAME = 'masarhub/synapse'
        GIT_BRANCH = 'develop'
        SKIP_DOCKER_LOGIN = 'false'
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
                    docker buildx rm multiarch-builder || true
                    docker buildx create --use --name multiarch-builder --driver docker-container
                    docker buildx inspect --bootstrap
                """
            }
        }

        stage('Get Version Info') {
            steps {
                script {
                    try {
                        env.COMMIT_HASH = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                        echo "✅ Commit Hash: ${env.COMMIT_HASH}"
                    } catch (Exception e) {
                        env.COMMIT_HASH = "build-${BUILD_NUMBER}"
                        echo "⚠️ Git unavailable, using build number: ${env.COMMIT_HASH}"
                    }
                    
                    // Ensure we have a valid version tag
                    if (!env.COMMIT_HASH || env.COMMIT_HASH.isEmpty()) {
                        env.COMMIT_HASH = "build-${BUILD_NUMBER}"
                        echo "🔄 Using fallback version: ${env.COMMIT_HASH}"
                    }
                }
            }
        }

        stage('Build and Push Docker Image') {
            steps {
                script {

                    if (env.SKIP_DOCKER_LOGIN == 'true') {
                        echo "Skipping Docker Hub login (testing mode)."
                    } else {
                        // Use the environment variable for credentials dynamically
                        withCredentials([
                            usernamePassword(
                                credentialsId: "${env.DOCKERHUB_CREDENTIALS}",
                                usernameVariable: 'DOCKER_USERNAME',
                                passwordVariable: 'DOCKER_PASSWORD'
                            )
                        ]) {
                            echo "Logging in to Docker Hub as ${DOCKER_USERNAME}"
                            def loginResult = sh(
                                script: 'echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin',
                                returnStatus: true
                            )
                            if (loginResult != 0) {
                                error "Docker Hub login failed. Check Jenkins credentials."
                            }
                            echo "✅ Successfully logged in to Docker Hub"
                        }
                    }

                    sh "docker pull ${BASE_IMAGE}"

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

                    echo "✅ Successfully built and pushed:"
                    echo "- ${IMAGE_NAME}:latest"
                    echo "- ${IMAGE_NAME}:${env.COMMIT_HASH}"
                }
            }
        }
    }

    post {
        always {
            script {
                echo "🧹 Cleaning up Docker resources"
                sh "docker buildx prune -f || true"
                sh "docker image prune -f || true"
                echo "Pipeline completed. Images pushed:"
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
            echo "❌ Build failed. Check logs above."
        }
    }
}
