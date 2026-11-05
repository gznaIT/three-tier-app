pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS_ID = 'Dockerhub'
        DOCKERHUB_USER = 'zainabdev'
        IMAGE_TAG = "${env.BRANCH_NAME ?: 'main'}-${BUILD_NUMBER}"
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

        stage('Build Backend Image') {
            steps {
                sh '''
                    docker build \
                    -t zainabdev/three-tier-app-backend:${IMAGE_TAG} \
                    ./backend
                '''
            }
        }

        stage('Build Frontend Image') {
            steps {
                sh '''
                    docker build \
                    -t zainabdev/three-tier-app-frontend:${IMAGE_TAG} \
                    ./frontend
                '''
            }
        }

        stage('Push Images to Docker Hub') {
            steps {
                sh '''
                    docker push zainabdev/three-tier-app-backend:${IMAGE_TAG}
                    docker push zainabdev/three-tier-app-frontend:${IMAGE_TAG}
                '''
            }
        }

        stage('Prepare Environment') {
            steps {
                sh '''
                    cat > .env <<EOF
BACKEND_IMAGE=zainabdev/three-tier-app-backend:${IMAGE_TAG}
FRONTEND_IMAGE=zainabdev/three-tier-app-frontend:${IMAGE_TAG}
EOF
                '''
            }
        }

        stage('Approval for Staging / Production') {
            when {
                anyOf {
                    branch 'stg'
                    branch 'prod'
                }
            }
            steps {
                input message: "Deploy to ${env.BRANCH_NAME} environment?",
                      ok: 'Deploy'
            }
        }

        stage('Deploy Application') {
            steps {
                sh '''
                    docker-compose --env-file .env pull
                    docker-compose --env-file .env up -d --remove-orphans
                '''
            }
        }

        stage('Verify Deployment') {
            steps {
                sh '''
                    docker-compose ps
                '''
            }
        }

        stage('Cleanup') {
            steps {
                sh '''
                    docker logout || true
                '''
            }
        }
    }

    post {
        success {
            echo "SUCCESS: ${env.BRANCH_NAME} deployed successfully using Docker Hub!"
        }

        failure {
            echo "FAILED: Pipeline failed. Check console output."
        }

        always {
            echo 'Jenkins pipeline execution finished.'
        }
    }
}


What I fixed
