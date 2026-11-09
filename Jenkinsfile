pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        timestamps()
        disableConcurrentBuilds()
    }

    environment {
        DOCKERHUB_CREDENTIALS_ID = 'Dockerhub'
        DOCKERHUB_USERNAME = 'zainabdev'
        IMAGE_TAG = "${BRANCH_NAME}-${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                echo "Building branch: ${BRANCH_NAME}"
                echo "Image tag: ${IMAGE_TAG}"
            }
        }

        stage('Validate Branch') {
            steps {
                script {
                    def allowedBranches = ['main', 'dev', 'stg', 'prod']

                    if (!allowedBranches.contains(env.BRANCH_NAME)) {
                        error("Unsupported branch: ${env.BRANCH_NAME}")
                    }
                }
            }
        }

        stage('Docker Hub Login') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: "${DOCKERHUB_CREDENTIALS_ID}",
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                        set -eu
                        echo "$DOCKER_PASS" | docker login \
                            --username "$DOCKER_USER" \
                            --password-stdin
                    '''
                }
            }
        }

        stage('Build Backend Image') {
            steps {
                sh '''
                    set -eu
                    docker build \
                        -t zainabdev/three-tier-app-backend:${IMAGE_TAG} \
                        ./backend
                '''
            }
        }

        stage('Build Frontend Image') {
            steps {
                sh '''
                    set -eu
                    docker build \
                        -t zainabdev/three-tier-app-frontend:${IMAGE_TAG} \
                        ./frontend
                '''
            }
        }

        stage('Push Images to Docker Hub') {
            steps {
                sh '''
                    set -eu

                    docker push zainabdev/three-tier-app-backend:${IMAGE_TAG}
                    docker push zainabdev/three-tier-app-frontend:${IMAGE_TAG}
                '''
            }
        }

        stage('Prepare Deployment Environment') {
            when {
                anyOf {
                    branch 'dev'
                    branch 'stg'
                    branch 'prod'
                }
            }
            steps {
                script {
                    def frontendPort
                    def backendPort

                    switch (env.BRANCH_NAME) {
                        case 'dev':
                            frontendPort = '3001'
                            backendPort = '5001'
                            break

                        case 'stg':
                            frontendPort = '3002'
                            backendPort = '5002'
                            break

                        case 'prod':
                            frontendPort = '3003'
                            backendPort = '5003'
                            break
                    }

                    env.COMPOSE_PROJECT = "three-tier-${env.BRANCH_NAME}"

                    writeFile file: '.env', text: """\
BACKEND_IMAGE=zainabdev/three-tier-app-backend:${env.IMAGE_TAG}
FRONTEND_IMAGE=zainabdev/three-tier-app-frontend:${env.IMAGE_TAG}
BACKEND_HOST_PORT=${backendPort}
FRONTEND_HOST_PORT=${frontendPort}
"""
                    echo "Backend image: zainabdev/three-tier-app-backend:${env.IMAGE_TAG}"
                    echo "Frontend image: zainabdev/three-tier-app-frontend:${env.IMAGE_TAG}"
                    echo "Compose project: ${env.COMPOSE_PROJECT}"
                    echo "Frontend host port: ${frontendPort}"
                    echo "Backend host port: ${backendPort}"
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
                    set -eu

                    echo "Validating Docker Compose configuration..."

                    docker compose \
                        --project-name "$COMPOSE_PROJECT" \
                        --env-file .env \
                        config -q

                    docker compose \
                        --project-name "$COMPOSE_PROJECT" \
                        --env-file .env \
                        config --format json |
                    python3 -c '
import json
import os
import sys

config = json.load(sys.stdin)

backend = config["services"]["backend"]["image"]
frontend = config["services"]["frontend"]["image"]

expected_backend = "zainabdev/three-tier-app-backend:" + os.environ["IMAGE_TAG"]
expected_frontend = "zainabdev/three-tier-app-frontend:" + os.environ["IMAGE_TAG"]

print("Backend image:", backend)
print("Frontend image:", frontend)

if backend != expected_backend:
    raise SystemExit("ERROR: Backend image tag mismatch")

if frontend != expected_frontend:
    raise SystemExit("ERROR: Frontend image tag mismatch")

print("Compose validation successful!")
'
                '''
            }
        }

        stage('Approval for Staging and Production') {
            when {
                anyOf {
                    branch 'stg'
                    branch 'prod'
                }
            }
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    input message: "Approve deployment of ${BRANCH_NAME} (${IMAGE_TAG})?",
                          ok: 'Deploy'
                }
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
                    set -eu

                    echo "Pulling Docker images..."

                    docker compose \
                        --project-name "$COMPOSE_PROJECT" \
                        --env-file .env \
                        pull

                    echo "Deploying application..."

                    docker compose \
                        --project-name "$COMPOSE_PROJECT" \
                        --env-file .env \
                        up -d --remove-orphans

                    echo "Deployment completed."
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
                    set -eu

                    docker compose \
                        --project-name "$COMPOSE_PROJECT" \
                        --env-file .env \
                        ps
                '''
            }
        }

        stage('Main Branch Deployment Skipped') {
            when {
                branch 'main'
            }
            steps {
                echo 'Main branch: images built and pushed successfully. Deployment intentionally skipped.'
            }
        }
    }

    post {
        always {
            sh '''
                docker logout || true
            '''
            echo "Pipeline finished for ${BRANCH_NAME}"
        }

        success {
            echo 'Pipeline completed successfully!'
        }

        failure {
            echo 'Pipeline failed. Check the first failed stage in Console Output.'
        }
    }
}
