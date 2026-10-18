pipeline {
    agent any

    environment {
        AWS_REGION = "us-east-1"
        AWS_ACCOUNT_ID = "913615820610"
        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        FRONTEND_IMAGE = "${ECR_REGISTRY}/frontend"
        BACKEND_IMAGE = "${ECR_REGISTRY}/backend"
        AWS_CREDENTIALS_ID = "Aws-key"
        GIT_REPO_URL = "https://github.com/gznaIT/three-tier-app.git"
    }

    stages {
        stage('Clone the Repo') {
            steps {
                git branch: 'main', url: "${GIT_REPO_URL}"
            }
        }

        stage('Build Docker Images') {
            steps {
                script {
                    def tag = "${BUILD_NUMBER}"
                    env.FRONTEND_TAG = "${FRONTEND_IMAGE}:${tag}"
                    env.BACKEND_TAG = "${BACKEND_IMAGE}:${tag}"

                    sh "docker build -t ${FRONTEND_TAG} ./frontend"
                    sh "docker build -t ${BACKEND_TAG} ./backend"
                }
            }
        }

        stage('Login to AWS ECR') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "${AWS_CREDENTIALS_ID}",
                    usernameVariable: 'AWS_ACCESS_KEY_ID',
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    sh '''
                        export AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID
                        export AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY
                        export AWS_DEFAULT_REGION=${AWS_REGION}
                        aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                    '''
                }
            }
        }

        stage('Push to AWS ECR') {
            steps {
                sh "docker push ${BACKEND_TAG}"
                sh "docker push ${FRONTEND_TAG}"
            }
        }

        stage('Update docker-compose.yml Tags') {
            steps {
                script {
                    sh "sed -i 's|image: .*backend.*|image: ${BACKEND_TAG}|g' docker-compose.yml"
                    sh "sed -i 's|image: .*frontend.*|image: ${FRONTEND_TAG}|g' docker-compose.yml"
                }
            }
        }

        stage('Push Changes to GitHub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: "Github",
                    usernameVariable: 'GH_USER',
                    passwordVariable: 'GH_TOKEN'
                )]) {
                    sh '''
                        git config --global user.email "jenkins@pipeline.com"
                        git config --global user.name "Jenkins CI"
                        git add docker-compose.yml
                        git commit -m "Auto-update docker-compose.yml with ECR tags for build ${BUILD_NUMBER}" || true
                        git push https://${GH_USER}:${GH_TOKEN}@github.com/gznaIT/three-tier-app.git main
                    '''
                }
            }
        }

        stage('Run application with Docker Compose') {
            steps {
                sh 'docker compose down'
                sh 'docker compose pull'
                sh 'docker compose up -d'
            }
        }

        stage('Clean up After Push') {
            steps {
                sh "docker rmi -f ${FRONTEND_TAG} || true"
                sh "docker rmi -f ${BACKEND_TAG} || true"
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully and app deployed using AWS ECR images!"
        }
        failure {
            echo "Pipeline failed. Check the logs for details."
        }
    }
}
