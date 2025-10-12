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
                    def COMMIT_HASH = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                    sh "docker build -t ${IMAGE_NAME}:latest -t ${IMAGE_NAME}:${COMMIT_HASH} -f docker/Dockerfile ."
                    // Save COMMIT_HASH in env for later stages
                    env.COMMIT_HASH = COMMIT_HASH
                }
            }
        }

        stage('Login to Docker Hub') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: "${DOCKERHUB_CREDENTIALS}", 
                                                     usernameVariable: 'DOCKER_USER', 
                                                     passwordVariable: 'DOCKER_PASS')]) {
                        sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    sh "docker push ${IMAGE_NAME}:latest"
                    sh "docker push ${IMAGE_NAME}:${COMMIT_HASH}"
                }
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
