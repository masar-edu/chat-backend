pipeline {
    agent any

    environment {
        GITHUB_CREDENTIALS = 'github-token'
        DOCKERHUB_CREDENTIALS = 'docker-hub-credentials-id'
        IMAGE_NAME = 'masarhub/synapse'
        GIT_BRANCH = 'develop' // branch to pull
    }

    stages {
        stage('Checkout from GitHub') {
            steps {
                git branch: "${GIT_BRANCH}",
                    url: 'https://github.com/masar-edu/chat-backend.git',
                    credentialsId: "${GITHUB_CREDENTIALS}"
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    COMMIT_HASH = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    sh "docker build -t ${IMAGE_NAME}:latest -t ${IMAGE_NAME}:${COMMIT_HASH} -f docker/Dockerfile ."
                }
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

        stage('Push Docker Image') {
            steps {
                sh "docker push ${IMAGE_NAME}:latest"
                sh "docker push ${IMAGE_NAME}:${COMMIT_HASH}"
            }
        }
    }

    post {
        always {
            echo "Cleaning up local Docker images"
            sh "docker rmi ${IMAGE_NAME}:latest || true"
            sh "docker rmi ${IMAGE_NAME}:${COMMIT_HASH} || true"
        }
    }
}
