// Jenkins declarative pipeline for small-team DevOps delivery.
// Stages: checkout, test, security scan, build, deploy, smoke test, notify.

pipeline {
    agent any

    environment {
        APP_IMAGE = "devops-app"
        REGISTRY   = "registry.example.internal"
        IMAGE_TAG  = "${env.BUILD_NUMBER}-${env.GIT_COMMIT?.take(7) ?: 'local'}"
        FULL_IMAGE = "${REGISTRY}/${APP_IMAGE}:${IMAGE_TAG}"
    }

    options {
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_AUTHOR = sh(script: 'git log -1 --format="%an"', returnStdout: true).trim()
                    env.GIT_SUBJECT = sh(script: 'git log -1 --format="%s"', returnStdout: true).trim()
                }
                echo "Branch: ${env.BRANCH_NAME} | Commit: ${env.GIT_COMMIT?.take(7)} | Author: ${env.GIT_AUTHOR}"
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                    python3 -m venv .venv
                    . .venv/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt
                    pip install pytest pytest-cov
                '''
            }
        }

        stage('Lint & Format Check') {
            steps {
                sh '''
                    . .venv/bin/activate
                    python -m flake8 . --max-line-length=100 --exclude=.venv,migrations || true
                '''
            }
        }

        stage('Unit Tests') {
            steps {
                sh '''
                    . .venv/bin/activate
                    pytest tests/ --tb=short --cov=. --cov-report=xml --cov-report=term-missing -q
                '''
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'test-results/*.xml'
                    publishHTML([
                        allowMissing: true,
                        alwaysLinkToLastBuild: false,
                        keepAll: true,
                        reportDir: 'htmlcov',
                        reportFiles: 'index.html',
                        reportName: 'Coverage Report'
                    ])
                }
            }
        }

        stage('Security Scan (Trivy)') {
            steps {
                script {
                    // Build a temporary image for scanning
                    sh "docker build -t ${APP_IMAGE}:scan-${env.BUILD_NUMBER} ."
                    // Run Trivy scan; fail on CRITICAL vulnerabilities
                    sh """
                        trivy image \
                            --exit-code 1 \
                            --severity CRITICAL \
                            --no-progress \
                            --format table \
                            ${APP_IMAGE}:scan-${env.BUILD_NUMBER} || \
                        (echo 'CRITICAL vulnerabilities found. Blocking deployment.' && exit 1)
                    """
                }
            }
            post {
                always {
                    sh "docker rmi ${APP_IMAGE}:scan-${env.BUILD_NUMBER} || true"
                }
            }
        }

        stage('Build & Push Image') {
            when {
                anyOf {
                    branch 'main'
                    branch 'release/*'
                }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'registry-creds',
                                                   usernameVariable: 'REG_USER',
                                                   passwordVariable: 'REG_PASS')]) {
                    sh """
                        echo "\$REG_PASS" | docker login ${REGISTRY} -u "\$REG_USER" --password-stdin
                        docker build -t ${FULL_IMAGE} .
                        docker push ${FULL_IMAGE}
                        docker tag ${FULL_IMAGE} ${REGISTRY}/${APP_IMAGE}:latest
                        docker push ${REGISTRY}/${APP_IMAGE}:latest
                        docker logout ${REGISTRY}
                    """
                }
            }
        }

        stage('Deploy to Staging') {
            when {
                branch 'main'
            }
            steps {
                sh """
                    IMAGE_TAG=${IMAGE_TAG} docker compose -f docker-compose.yml pull app
                    IMAGE_TAG=${IMAGE_TAG} docker compose -f docker-compose.yml up -d app
                    echo "Deployed image ${FULL_IMAGE} to staging"
                """
            }
        }

        stage('Smoke Test') {
            when {
                branch 'main'
            }
            steps {
                sh '''
                    sleep 10
                    curl -sf http://localhost:8000/health | python3 -c "
import sys, json
data = json.load(sys.stdin)
assert data.get('status') == 'ok', f'Health check failed: {data}'
print('Smoke test passed:', data)
"
                '''
            }
        }
    }

    post {
        success {
            echo "Pipeline succeeded: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
        }
        failure {
            echo "Pipeline FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}"
            script {
                // Rollback on main branch failures
                if (env.BRANCH_NAME == 'main') {
                    sh "docker compose -f docker-compose.yml up -d --no-deps app || true"
                    echo "Rollback attempted."
                }
            }
        }
        always {
            cleanWs(cleanWhenFailure: false)
        }
    }
}
