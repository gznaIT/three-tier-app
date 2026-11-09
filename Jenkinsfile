```groovy
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        DOCKERHUB_USER = 'zainabdev'
        DOCKERHUB_CREDENTIALS_ID = 'Dockerhub'

        BACKEND_IMAGE = 'zainabdev/three-tier-app-backend'
        FRONTEND_IMAGE = 'zainabdev/three-tier-app-frontend'

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
                withCredentials([
                    usernamePassword(
                        credentialsId: "${DOCKERHUB_CREDENTIALS_ID}",
                        usernameVariable: 'DOCKERHUB_USER_VAR',
                        passwordVariable: 'DOCKERHUB_PASS'
                    )
                ]) {
                    sh '''
                        printf '%s' "$DOCKERHUB_PASS" |
                        docker login --username "$DOCKERHUB_USER_VAR" --password-stdin
                    '''
                }
            }
        }

        stage('Build Backend Image') {
            steps {
                sh '''
                    docker build -t ${BACKEND_IMAGE}:${IMAGE_TAG} ./backend
                '''
            }
        }

        stage('Build Frontend Image') {
            steps {
                sh '''
                    docker build -t ${FRONTEND_IMAGE}:${IMAGE_TAG} ./frontend
                '''
            }
        }

        stage('Push Images to Docker Hub') {
            steps {
                sh '''
                    docker push ${BACKEND_IMAGE}:${IMAGE_TAG}
                    docker push ${FRONTEND_IMAGE}:${IMAGE_TAG}
                '''
            }
        }

        stage('Prepare Environment') {
            steps {
                script {
                    def ports = [
                        dev:  [frontend: '3001', backend: '5001'],
                        stg:  [frontend: '3002', backend: '5002'],
                        prod: [frontend: '3003', backend: '5003']
                    ]

                    if (env.BRANCH_NAME == 'main') {
                        echo 'Main branch: images pushed; deployment skipped.'
                    } else {
                        if (!ports.containsKey(env.BRANCH_NAME)) {
                            error("Unsupported deployment branch: ${env.BRANCH_NAME}")
                        }

                        def selected = ports[env.BRANCH_NAME]

                        writeFile(
                            file: '.env',
                            text: """BACKEND_IMAGE=${env.BACKEND_IMAGE}:${env.IMAGE_TAG}
FRONTEND_IMAGE=${env.FRONTEND_IMAGE}:${env.IMAGE_TAG}
BACKEND_HOST_PORT=${selected.backend}
FRONTEND_HOST_PORT=${selected.frontend}
"""
                        )

                        env.COMPOSE_PROJECT = "three-tier-${env.BRANCH_NAME}"

                        echo "Backend image: ${env.BACKEND_IMAGE}:${env.IMAGE_TAG}"
                        echo "Frontend image: ${env.FRONTEND_IMAGE}:${env.IMAGE_TAG}"
                        echo "Compose project: ${env.COMPOSE_PROJECT}"
                        echo "Frontend port: ${selected.frontend}"
                        echo "Backend port: ${selected.backend}"
                    }
                }
            }
        }

        stage('Validate Compose Configuration') {
            when {
                anyOf {
                    branch 'dev'
                    branch 'stg'
                    branch 'prod'
                }
            }
            steps {
                sh '''
                    echo "Checking Docker Compose configuration..."

                    docker compose \
                      --project-name "$COMPOSE_PROJECT" \
                      --env-file .env config -q

                    echo "Resolved image references:"
                    docker compose \
                      --project-name "$COMPOSE_PROJECT" \
                      --env-file .env config --images

                    docker compose \
                      --project-name "$COMPOSE_PROJECT" \
                      --env-file .env config --images > resolved-images.txt

                    grep -Fx "${BACKEND_IMAGE}:${IMAGE_TAG}" resolved-images.txt
                    grep -Fx "${FRONTEND_IMAGE}:${IMAGE_TAG}" resolved-images.txt

                    echo "Compose configuration and image tags verified."
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
            options {
                timeout(time: 30, unit: 'MINUTES')
            }
            steps {
                input(
                    message: "Approve deployment to ${env.BRANCH_NAME}?",
                    ok: 'Deploy'
                )
            }
        }

        stage('Deploy Application') {
            when {
                anyOf {
                    branch 'dev'
                    branch 'stg'
                    branch 'prod'
                }
            }
            steps {
                sh '''
                    docker compose \
                      --project-name "$COMPOSE_PROJECT" \
                      --env-file .env pull

                    docker compose \
                      --project-name "$COMPOSE_PROJECT" \
                      --env-file .env up -d --remove-orphans
                '''
            }
        }

        stage('Verify Deployment') {
            when {
                anyOf {
                    branch 'dev'
                    branch 'stg'
                    branch 'prod'
                }
            }
            steps {
                sh '''
                    docker compose \
                      --project-name "$COMPOSE_PROJECT" \
                      --env-file .env ps
                '''
            }
        }

    }

    post {
        success {
            echo "SUCCESS: ${env.BRANCH_NAME} pipeline completed using Docker Hub!"
        }

        failure {
            echo "FAILED: Pipeline failed. Check Console Output."
        }

        always {
            sh 'docker logout || true'
            echo 'Jenkins pipeline execution finished.'
        }
    }
}
```
