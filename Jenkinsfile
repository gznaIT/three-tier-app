@Library('sharedLib') _

pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
    }

    environment {
        DOCKERHUB_CREDENTIALS_ID = 'Dockerhub'
        DOCKERHUB_USER = 'zainabdev'
        IMAGE_TAG = "main-${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout Source') {
            steps {
                checkoutSource()
            }
        }

        stage('Docker Hub Login') {
            steps {
                dockerLogin(env.DOCKERHUB_CREDENTIALS_ID)
            }
        }

        stage('Build Backend and Frontend Images') {
            steps {
                buildImages(
                    env.DOCKERHUB_USER,
                    './backend',
                    './frontend',
                    env.IMAGE_TAG
                )
            }
        }

        stage('Push Images to Docker Hub') {
            steps {
                pushImages()
            }
        }

        stage('Cleanup') {
            steps {
                cleanupImages()
            }
        }
    }

    post {
        success {
            echo "SUCCESS: Both Docker images were pushed to Docker Hub."
            echo "Backend: zainabdev/three-tier-app-backend:${env.IMAGE_TAG}"
            echo "Frontend: zainabdev/three-tier-app-frontend:${env.IMAGE_TAG}"
        }

        failure {
            echo 'FAILED: Check the Jenkins Console Output.'
        }

        always {
            sh 'docker logout || true'
            echo 'Pipeline execution finished.'
        }
    }
}
