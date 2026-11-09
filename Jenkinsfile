pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
    }

    environment {
        DOCKERHUB_USER = 'zainabdev'
        DOCKERHUB_CREDENTIALS_ID = 'Dockerhub'

        // Repository names - deliberately different from Compose variables
        BACKEND_IMAGE_REPO = 'zainabdev/three-tier-app-backend'
        FRONTEND_IMAGE_REPO = 'zainabdev/three-tier-app-frontend'

        IMAGE_TAG = "${env.BRANCH_NAME ?: 'main'}-${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout Source') {
            steps {
                checkout scm
                echo "Building branch: ${env.BRANCH_NAME}"
                echo "Image tag: ${env.IMAGE_TAG}"
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
                        usernameVariable: 'DOCKERHUB_USER_VAR',
                        passwordVariable: 'DOCKERHUB_PASS'
                    )
                ]) {
                    sh '''
                        set -eu
                        printf '%s' "$DOCKERHUB_PASS" |
                        docker login \
                          --username "$DOCKERHUB_USER_VAR" \
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
                      -t ${BACKEND_IMAGE_REPO}:${IMAGE_TAG} \
                      ./backend
                '''
            }
        }

        stage('Build Frontend Image') {
            steps {
                sh '''
                    set -eu
                    docker build \
                      -t ${FRONTEND_IMAGE_REPO}:${IMAGE_TAG} \
                      ./frontend
                '''
            }
        }

        stage('Push Images to Docker Hub') {
            steps {
                sh '''
                    set -eu
                    docker push ${BACKEND_IMAGE_REPO}:${IMAGE_TAG}
                    docker push ${FRONTEND_IMAGE_REPO}:${IMAGE_TAG}
                '''
            }
        }

        stage('Prepare Environment') {
            steps {
                script {
                    def ports = [
                        dev: [
                            frontend: '3001',
                            backend: '5001'
                        ],
                        stg: [
                            frontend: '3002',
                            backend: '5002'
                        ],
                        prod: [
                            frontend: '3003',
                            backend: '5003'
                        ]
                    ]

                    if (env.BRANCH_NAME == 'main') {
                        echo 'Main branch: images pushed; deployment skipped.'
                    } else {
                        def selected = ports[env.BRANCH_NAME]

                        def backendFullImage =
                            "${env.BACKEND_IMAGE_REPO}:${env.IMAGE_TAG}"

                        def frontendFullImage =
                            "${env.FRONTEND_IMAGE_REPO}:${env.IMAGE_TAG}"

                        writeFile(
                            file: '.env',
                            text: """BACKEND_IMAGE=${backendFullImage}
FRONTEND_IMAGE=${frontendFullImage}
BACKEND_HOST_PORT=${selected.backend}
FRONTEND_HOST_PORT=${selected.frontend}
"""
                        )

                        env.COMPOSE_PROJECT =
                            "three-tier-${env.BRANCH_NAME}"

                        echo "Backend image: ${backendFullImage}"
                        echo "Frontend image: ${frontendFullImage}"
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

expected_backend = (
    "zainabdev/three-tier-app-backend:" + os.environ["IMAGE_TAG"]
)
expected_frontend = (
    "zainabdev/three-tier-app-frontend:" + os.environ["IMAGE_TAG"]
)

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

        stage('Approval for Staging / Production') {
            when {
                anyOf {
                    branch 'stg'
                    branch 'prod'
                }
            }
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    input(
                        message: "Approve deployment to ${env.BRANCH_NAME}?",
                        ok: 'Deploy'
                    )
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

                    echo "Pulling Docker Hub images..."

                    docker compose \
                      --project-name "$COMPOSE_PROJECT" \
                      --env-file .env \
                      pull

                    echo "Deploying application..."

                    docker compose \
                      --project-name "$COMPOSE_PROJECT" \
                      --env-file .env \
                      up -d --remove-orphans
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
                echo 'Main branch: images built and pushed. Deployment intentionally skipped.'
            }
        }
    }

    post {
        success {
            echo "SUCCESS: ${env.BRANCH_NAME} pipeline completed using Docker Hub!"
        }

        failure {
            echo "FAILED: ${env.BRANCH_NAME} pipeline. Check Console Output."
        }

        always {
            sh 'docker logout || true'
            echo 'Jenkins pipeline execution finished.'
        }
    }
}
