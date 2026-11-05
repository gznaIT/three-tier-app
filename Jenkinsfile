```groovy
pipeline {
    agent any

    environment {
        // Docker Hub credentials configured in Jenkins
        DOCKERHUB_CREDENTIALS_ID = 'dockerhub-credentials'
        DOCKERHUB_USER = 'zainabdev'

        // Dynamic image tags
        BRANCH_NAME = "${env.BRANCH_NAME ?: 'main'}"
        IMAGE_TAG = "${BRANCH_NAME}-${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout Source') {
            steps {
                checkout scm
            }
        }

        stage('Docker Hub Login') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${DOCKERHUB_CREDENTIALS_ID}",
                    usernameVariable: 'DOCKERHUB_USER_VAR',
                    passwordVariable: 'DOCKERHUB_PASS'
                )]) {
                    sh '''
                        echo "$DOCKERHUB_PASS" | docker login \
                        -u "$DOCKERHUB_USER_VAR" --password-stdin
                    '''
                }
            }
        }

        stage('Build & Tag Images') {
            steps {
                script {
                    env.BACKEND_TAG_DH = "${DOCKERHUB_USER}/three-tier-app-backend:${IMAGE_TAG}"
                    env.FRONTEND_TAG_DH = "${DOCKERHUB_USER}/three-tier-app-frontend:${IMAGE_TAG}"

                    sh "docker build -t ${env.BACKEND_TAG_DH} ./backend"
                    sh "docker build -t ${env.FRONTEND_TAG_DH} ./frontend"
                }
            }
        }

        stage('Push Images to Docker Hub') {
            steps {
                sh """
                    docker push ${env.BACKEND_TAG_DH}
                    docker push ${env.FRONTEND_TAG_DH}
                """
            }
        }

        stage('Prepare .env for Compose') {
            steps {
                script {
                    writeFile file: '.env', text: """BACKEND_IMAGE=${env.BACKEND_TAG_DH}
FRONTEND_IMAGE=${env.FRONTEND_TAG_DH}
"""
                }
            }
        }

        stage('Approval for Staging / Prod Deploy') {
            when {
                anyOf {
                    branch 'stg'
                    branch 'prod'
                }
            }
            steps {
                input message: "Deploy to ${BRANCH_NAME} environment?",
                      ok: "Yes, Deploy"
            }
        }

        stage('Deploy Environment') {
            steps {
                sh """
                    docker-compose --env-file .env down
                    docker-compose --env-file .env pull
                    docker-compose --env-file .env up -d --remove-orphans
                """
            }
        }

        stage('Cleanup Local Images') {
            steps {
                sh """
                    docker rmi ${env.BACKEND_TAG_DH} ${env.FRONTEND_TAG_DH} || true
                    docker logout
                """
            }
        }
    }

    post {
        success {
            echo "SUCCESS: ${BRANCH_NAME} environment deployed using Docker Hub images!"
        }

        failure {
            echo "FAILED: Deployment failed for ${BRANCH_NAME}. Check logs."
        }
    }
}
```
