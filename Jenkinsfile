pipeline {
    agent any

    tools {
        nodejs 'node21'
    }

    environment {
        DOCKERHUB_USERNAME = "biswajit7815"
        BACKEND_IMAGE = "blood-backend"
        FRONTEND_IMAGE = "blood-frontend"
        IMAGE_TAG = "${BUILD_NUMBER}"

        SCANNER_HOME = tool 'sonar-scanner'

        BACKEND_CONTAINER = 'blood-backend'
        FRONTEND_CONTAINER = 'blood-frontend'

        BACKEND_PORT = '5000'
        EC2_PUBLIC_IP = "65.2.189.152"
    }

    stages {

        // =========================
        // CLEANUP & CHECKOUT
        // =========================
        stage('cleanup & checkout') {
            steps {
                cleanWs()
                checkout scm
                echo "Code checkout complete - Build #${BUILD_NUMBER}"
            }
        }

        // =========================
        // INSTALL DEPENDENCIES
        // =========================
        stage('install dependencies') {
            parallel {

                stage('Backend install') {
                    steps {
                        dir('backend') {
                            sh 'npm install'
                        }
                    }
                }

                stage('Frontend install') {
                    steps {
                        dir('frontend') {
                            sh 'npm install'
                        }
                    }
                }
            }
        }

        // =========================
        // SECURITY SCAN
        // =========================
        stage('security scan') {
            parallel {

                // OWASP Dependency Check
                stage('OWASP Dependency Check') {
                    steps {

                        sh 'mkdir -p reports/owasp'

                        dependencyCheck(
                            additionalArguments: '''
                                --scan backend/
                                --scan frontend/
                                --format HTML
                                --format XML
                                --out reports/owasp/
                                --disableAssembly
                                --disableYarnAudit
                                --disableNodeAudit
                                --prettyPrint
                            ''',
                            odcInstallation: 'DP-Check'
                        )

                        dependencyCheckPublisher(
                            pattern: 'reports/owasp/dependency-check-report.xml',
                            failedTotalCritical: 20,
                            unstableTotalCritical: 5
                        )
                    }
                }

                // Trivy FS Scan
                stage('Trivy FS Scan') {
                    steps {

                        sh '''
                            mkdir -p reports/trivy

                            trivy fs . \
                                --exit-code 0 \
                                --severity HIGH,CRITICAL \
                                --format table \
                                -o reports/trivy/fs-scan.txt

                            cat reports/trivy/fs-scan.txt
                        '''
                    }
                }
            }
        }

        // =========================
        // SONARQUBE
        // =========================
        stage('SonarQube Analysis') {
            steps {

                withSonarQubeEnv('sonar-server') {

                    sh """
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=blood-bank-management \
                        -Dsonar.projectName=blood-bank-management \
                        -Dsonar.sources=backend,frontend
                    """
                }
            }
        }

        // =========================
        // BUILD DOCKER IMAGES
        // =========================
        stage('Build Docker Images') {
            steps {

                echo "Building backend image..."

                sh """
                    docker build \
                        -t ${DOCKERHUB_USERNAME}/${BACKEND_IMAGE}:${IMAGE_TAG} \
                        -t ${DOCKERHUB_USERNAME}/${BACKEND_IMAGE}:latest \
                        -f backend/Dockerfile \
                        ./backend
                """

                echo "Building frontend image..."

                sh """
                    docker build \
                        --build-arg VITE_API_PATH=http://${EC2_PUBLIC_IP}:${BACKEND_PORT} \
                        -t ${DOCKERHUB_USERNAME}/${FRONTEND_IMAGE}:${IMAGE_TAG} \
                        -t ${DOCKERHUB_USERNAME}/${FRONTEND_IMAGE}:latest \
                        -f frontend/Dockerfile \
                        ./frontend
                """
            }
        }

        // =========================
        // TRIVY IMAGE SCAN
        // =========================
        stage('Trivy Image Scan') {
            steps {

                sh """
                    trivy image \
                        --exit-code 0 \
                        --severity HIGH,CRITICAL \
                        --format table \
                        -o reports/trivy/backend-image-scan.txt \
                        ${DOCKERHUB_USERNAME}/${BACKEND_IMAGE}:latest

                    trivy image \
                        --exit-code 0 \
                        --severity HIGH,CRITICAL \
                        --format table \
                        -o reports/trivy/frontend-image-scan.txt \
                        ${DOCKERHUB_USERNAME}/${FRONTEND_IMAGE}:latest
                """
            }
        }

        // =========================
        // PUSH TO DOCKER HUB
        // =========================
        stage('Push to Docker Hub') {
            steps {

                script {

                    withCredentials([
                        usernamePassword(
                            credentialsId: 'docker-hub-creds',
                            passwordVariable: 'DOCKER_PASS',
                            usernameVariable: 'DOCKER_USER'
                        )
                    ]) {

                        sh "echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin"

                        sh "docker push ${DOCKERHUB_USERNAME}/${BACKEND_IMAGE}:${IMAGE_TAG}"
                        sh "docker push ${DOCKERHUB_USERNAME}/${BACKEND_IMAGE}:latest"

                        sh "docker push ${DOCKERHUB_USERNAME}/${FRONTEND_IMAGE}:${IMAGE_TAG}"
                        sh "docker push ${DOCKERHUB_USERNAME}/${FRONTEND_IMAGE}:latest"

                        sh "docker logout"
                    }
                }
            }
        }

        // =========================
        // DEPLOY
        // =========================
        stage('Deploy') {
            steps {

                script {

                    withCredentials([
                        string(credentialsId: 'MONGO_URI', variable: 'MONGO_URI'),
                        string(credentialsId: 'JWT_SECRET', variable: 'JWT_SECRET')
                    ]) {

                        echo "Stopping old containers..."

                        sh "docker stop ${BACKEND_CONTAINER} || true"
                        sh "docker stop ${FRONTEND_CONTAINER} || true"

                        sh "docker rm ${BACKEND_CONTAINER} || true"
                        sh "docker rm ${FRONTEND_CONTAINER} || true"

                        echo "Creating Docker network..."

                        sh "docker network create blood-network || true"

                        echo "Starting backend container..."

                        sh """
                            docker run -d \
                                --name ${BACKEND_CONTAINER} \
                                --network blood-network \
                                --restart unless-stopped \
                                -p ${BACKEND_PORT}:${BACKEND_PORT} \
                                -e MONGO_URI="${MONGO_URI}" \
                                -e JWT_SECRET="${JWT_SECRET}" \
                                ${DOCKERHUB_USERNAME}/${BACKEND_IMAGE}:${IMAGE_TAG}
                        """

                        echo "Starting frontend container..."

                        sh """
                            docker run -d \
                                --name ${FRONTEND_CONTAINER} \
                                --network blood-network \
                                --restart unless-stopped \
                                -p 80:80 \
                                ${DOCKERHUB_USERNAME}/${FRONTEND_IMAGE}:${IMAGE_TAG}
                        """

                        echo "Waiting for containers..."

                        sh "sleep 20"

                        echo "Container status..."

                        sh "docker ps --filter 'name=${BACKEND_CONTAINER}'"
                        sh "docker ps --filter 'name=${FRONTEND_CONTAINER}'"

                        echo "Backend health check..."

                        sh "curl -sf http://localhost:${BACKEND_PORT}/health && echo 'Backend Healthy'"

                        echo "Frontend health check..."

                        sh "curl -sf http://localhost && echo 'Frontend Healthy'"

                        echo "Application Live: http://${EC2_PUBLIC_IP}"
                    }
                }
            }
        }

        // =========================
        // CLEANUP OLD IMAGES
        // =========================
        stage('Cleanup Old Images') {
            steps {

                echo "Cleaning dangling images..."

                sh "docker image prune -f"

                echo "Removing old backend images..."

                sh """
                    docker images ${DOCKERHUB_USERNAME}/${BACKEND_IMAGE} --format "{{.Tag}}" \
                        | grep -v "latest" \
                        | grep -v "${IMAGE_TAG}" \
                        | xargs -r -I {} docker rmi ${DOCKERHUB_USERNAME}/${BACKEND_IMAGE}:{} || true
                """

                echo "Removing old frontend images..."

                sh """
                    docker images ${DOCKERHUB_USERNAME}/${FRONTEND_IMAGE} --format "{{.Tag}}" \
                        | grep -v "latest" \
                        | grep -v "${IMAGE_TAG}" \
                        | xargs -r -I {} docker rmi ${DOCKERHUB_USERNAME}/${FRONTEND_IMAGE}:{} || true
                """
            }
        }
    }

    post {

        always {

            archiveArtifacts artifacts: 'reports/**/*', allowEmptyArchive: true

            sh "docker logout || true"
        }

        success {
            echo "Build #${BUILD_NUMBER} deployed successfully - http://${EC2_PUBLIC_IP}"
        }

        failure {
            echo "Build #${BUILD_NUMBER} failed - ${BUILD_URL}console"
        }

        cleanup {

            cleanWs(
                cleanWhenSuccess: true,
                cleanWhenFailure: false,
                cleanWhenAborted: true
            )
        }
    }
}