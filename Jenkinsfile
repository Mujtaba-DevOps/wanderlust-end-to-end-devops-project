pipeline {

    agent any

    options {
        timestamps()
        skipStagesAfterUnstable()
    }

    environment {
        SONAR_HOME = tool 'sonar'
        DC_HOME = tool 'dc'

        DOCKER_BACKEND_IMAGE = "mujtabas/wanderlust-backend:${BUILD_NUMBER}"
        DOCKER_FRONTEND_IMAGE = "mujtabas/wanderlust-frontend:${BUILD_NUMBER}"
    }

    stages {

        stage('Clone Code') {
            steps {
                cleanWs()

                git branch: 'main',
                    url: 'https://github.com/Mujtaba-DevOps/wanderlust-end-to-end-devops-project.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                cd backend
                npm install

                cd ../frontend
                npm install
                '''
            }
        }

        stage('Verify Tools') {
            steps {
                sh '''
                java -version
                node -v
                npm -v
                docker --version
                kubectl version --client
                trivy --version
                ${SONAR_HOME}/bin/sonar-scanner --version
                '''
            }
        }

        stage('SonarQube Scan') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''
                    ${SONAR_HOME}/bin/sonar-scanner \
                    -Dsonar.projectKey=wanderlust \
                    -Dsonar.projectName=wanderlust \
                    -Dsonar.sources=. \
                    -Dsonar.sourceEncoding=UTF-8 \
                    -Dsonar.exclusions=**/node_modules/**,**/dist/**,**/build/**,**/coverage/**,**/*.png,**/*.jpg,**/*.jpeg,**/*.gif,**/*.svg,**/*.webp
                    '''
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'nvd-api-key',
                        variable: 'NVD_API_KEY'
                    )
                ]) {
                    sh '''
                    mkdir -p dependency-check-report

                    DC_SCRIPT=$(find "$DC_HOME" -name dependency-check.sh | head -1)

                    chmod +x "$DC_SCRIPT"

                    "$DC_SCRIPT" \
                    --project wanderlust \
                    --scan . \
                    --format HTML \
                    --format XML \
                    --out dependency-check-report \
                    --disableYarnAudit || true
                    '''
                }
            }
        }

        stage('Publish Dependency Check Report') {
            steps {
                dependencyCheckPublisher(
                    pattern: 'dependency-check-report/dependency-check-report.xml'
                )
            }
        }

        stage('Sonar Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Trivy File System Scan') {
            steps {
                sh '''
                mkdir -p trivy-report

                trivy fs . \
                --format table \
                --output trivy-report/trivy-fs-report.txt || true
                '''
            }
        }

        stage('Build Docker Images') {
            steps {
                sh '''
                docker build -t ${DOCKER_BACKEND_IMAGE} ./backend
                docker build -t ${DOCKER_FRONTEND_IMAGE} ./frontend
                '''
            }
        }

        stage('Push Docker Images') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

                    docker push ${DOCKER_BACKEND_IMAGE}
                    docker push ${DOCKER_FRONTEND_IMAGE}

                    docker logout
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                kubectl apply -f kubernetes/

                kubectl rollout status deployment/backend-deployment -n wanderlust
                kubectl rollout status deployment/frontend-deployment -n wanderlust
                kubectl rollout status deployment/mongo-deployment -n wanderlust
                kubectl rollout status deployment/redis-deployment -n wanderlust

                kubectl get all -n wanderlust
                '''
            }
        }
    }

    post {

        always {
            archiveArtifacts artifacts: '''
dependency-check-report/**
trivy-report/**
''',
            fingerprint: true,
            allowEmptyArchive: true

            cleanWs()
        }

        success {
            echo 'Pipeline Completed Successfully'
        }

        failure {
            echo 'Pipeline Failed'
        }
    }
}
