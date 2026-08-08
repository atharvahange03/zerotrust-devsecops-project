pipeline {
    agent any

    environment {
        AWS_ACCOUNT_ID = "848005667877"
        AWS_REGION = "us-east-1"
        ECR_REGISTRY = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        IMAGE_NAME_BACKEND = "${ECR_REGISTRY}/ztso-backend"
        IMAGE_NAME_FRONTEND = "${ECR_REGISTRY}/ztso-frontend"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.IMAGE_TAG = sh(
                        script: 'git rev-parse --short HEAD',
                        returnStdout: true
                    ).trim()
                }
                echo "Building commit: ${env.IMAGE_TAG}"
            }
        }

        stage('Gitleaks - Secret Scan') {
            steps {
                sh 'rm -rf .scannerwork'
                sh 'rm -f gitleaks-report.json trivy-backend-report.json trivy-frontend-report.json trivy-fs-report.json trivy-backend-high-report.json trivy-frontend-high-report.json'
                sh '''
                    gitleaks detect \
                        --source . \
                        --no-git \
                        --verbose \
                        --report-format json \
                        --report-path /tmp/gitleaks-report.json
                '''
            }
            post {
                always {
                    sh 'cp /tmp/gitleaks-report.json gitleaks-report.json || true'
                    archiveArtifacts artifacts: 'gitleaks-report.json', allowEmptyArchive: true
                }
            }
        }

        stage('SonarQube - SAST') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh "${tool 'sonarqube-scanner'}/bin/sonar-scanner"
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Trivy - Filesystem Scan') {
            steps {
                sh '''
                    trivy fs --severity HIGH,CRITICAL \
                        --format json \
                        --output trivy-fs-report.json \
                        .
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-fs-report.json', allowEmptyArchive: true
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh """
                    docker build -t ${IMAGE_NAME_BACKEND}:${IMAGE_TAG} ./backend
                    docker build -t ${IMAGE_NAME_FRONTEND}:${IMAGE_TAG} ./frontend
                """
            }
        }

        stage('Trivy - Image Scan') {
            steps {
                sh """
                    trivy image --severity CRITICAL --exit-code 1 --ignore-unfixed \
                        --format json --output trivy-backend-report.json \
                        ${IMAGE_NAME_BACKEND}:${IMAGE_TAG}

                    trivy image --severity CRITICAL --exit-code 1 --ignore-unfixed \
                        --format json --output trivy-frontend-report.json \
                        ${IMAGE_NAME_FRONTEND}:${IMAGE_TAG}

                    trivy image --severity HIGH --exit-code 0 --ignore-unfixed \
                        --format json --output trivy-backend-high-report.json \
                        ${IMAGE_NAME_BACKEND}:${IMAGE_TAG}

                    trivy image --severity HIGH --exit-code 0 --ignore-unfixed \
                        --format json --output trivy-frontend-high-report.json \
                        ${IMAGE_NAME_FRONTEND}:${IMAGE_TAG}
                """
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-backend-report.json, trivy-frontend-report.json, trivy-backend-high-report.json, trivy-frontend-high-report.json', allowEmptyArchive: true
                }
            }
        }

        stage('ECR Push') {
            steps {
                sh """
                    aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
                    docker push ${IMAGE_NAME_BACKEND}:${IMAGE_TAG}
                    docker push ${IMAGE_NAME_FRONTEND}:${IMAGE_TAG}
                """
            }
        }

        stage('Cosign - Image Sign') {
            steps {
                withCredentials([
                    file(credentialsId: 'cosign-private-key', variable: 'COSIGN_KEY'),
                    string(credentialsId: 'cosign-password', variable: 'COSIGN_PASSWORD')
                ]) {
                    sh """
                        cosign sign --key \$COSIGN_KEY \
                            -a "pipeline=jenkins" \
                            -a "commit=${IMAGE_TAG}" \
                            ${IMAGE_NAME_BACKEND}:${IMAGE_TAG} --yes
                        cosign sign --key \$COSIGN_KEY \
                            -a "pipeline=jenkins" \
                            -a "commit=${IMAGE_TAG}" \
                            ${IMAGE_NAME_FRONTEND}:${IMAGE_TAG} --yes
                    """
                }
            }
        }

        stage('Update Image Tag') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'gitCredentials',
                    usernameVariable: 'GIT_USER',
                    passwordVariable: 'GIT_TOKEN'
                )]) {
                    sh """
                        git config user.email "jenkins@ztso.local"
                        git config user.name "Jenkins"
                        sed -i 's|tag:.*|tag: "${IMAGE_TAG}"|g' k8s/argocd/values-override.yaml
                        git add k8s/argocd/values-override.yaml
                        git diff --staged --quiet || git commit -m "Update image tag to ${IMAGE_TAG} [skip ci]"
                        git push https://\${GIT_USER}:\${GIT_TOKEN}@github.com/atharvahange03/zerotrust-devsecops-project.git HEAD:ztso-devops-local-setup
                    """
                }
            }
        }
    }

    post {
        always {
            sh """
                docker rmi ${IMAGE_NAME_BACKEND}:${IMAGE_TAG} || true
                docker rmi ${IMAGE_NAME_FRONTEND}:${IMAGE_TAG} || true
                docker image prune -f || true
            """
        }
        success {
            withCredentials([string(credentialsId: 'slack-webhook-jenkins', variable: 'SLACK_WEBHOOK')]) {
                sh "curl -s -X POST -H 'Content-type: application/json' --data '{\"text\":\"✅ Pipeline Passed - Job: ${env.JOB_NAME} Build: ${env.BUILD_NUMBER} Commit: ${IMAGE_TAG}\"}' \$SLACK_WEBHOOK"
            }
            mail(
                to: 'media.apexmedia@gmail.com',
                subject: "✅ PASSED - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "Pipeline passed.\nCommit: ${IMAGE_TAG}\nView: ${env.BUILD_URL}"
            )
        }
        failure {
            withCredentials([string(credentialsId: 'slack-webhook-jenkins', variable: 'SLACK_WEBHOOK')]) {
                sh "curl -s -X POST -H 'Content-type: application/json' --data '{\"text\":\"🔴 Pipeline Failed - Job: ${env.JOB_NAME} Build: ${env.BUILD_NUMBER} Commit: ${IMAGE_TAG}\"}' \$SLACK_WEBHOOK"
            }
            mail(
                to: 'media.apexmedia@gmail.com',
                subject: "🔴 FAILED - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: "Pipeline failed.\nCommit: ${IMAGE_TAG}\nView: ${env.BUILD_URL}"
            )
        }
    }
}
