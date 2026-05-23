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
                            unstableTotalCritical: 20
                        )
                    }
                }

                // Trivy FS Scan karta he.....
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

    // HAMESHA CHALEGA - pass ho ya fail
    always {
        archiveArtifacts artifacts: 'reports/**/', allowEmptyArchive: true
        sh "docker logout || true"
    }

    // SIRF SUCCESS PAR
    success {
        emailext(
            to:       'biswajitbehera1868@gmail.com',
            subject:  "Build #${BUILD_NUMBER} - ${JOB_NAME} - SUCCESS",
            mimeType: 'text/html',
            attachmentsPattern: 'reports/trivy/*.txt',
            body:     """
                <html>
                <body style="font-family: Arial; padding: 20px; background-color: #f4f4f4;">

                    <div style="background-color: white; padding: 30px; border-radius: 8px; border-left: 5px solid green;">

                        <h2 style="color: green;">Build Successfully Deploy Ho Gaya!</h2>

                        <table border="1" cellpadding="10" cellspacing="0" width="100%">
                            <tr style="background-color: #f9f9f9;">
                                <td><b>Project</b></td>
                                <td>${JOB_NAME}</td>
                            </tr>
                            <tr>
                                <td><b>Build Number</b></td>
                                <td>#${BUILD_NUMBER}</td>
                            </tr>
                            <tr style="background-color: #f9f9f9;">
                                <td><b>Status</b></td>
                                <td style="color: green;"><b>SUCCESS</b></td>
                            </tr>
                            <tr>
                                <td><b>Duration</b></td>
                                <td>${currentBuild.durationString}</td>
                            </tr>
                            <tr style="background-color: #f9f9f9;">
                                <td><b>Application URL</b></td>
                                <td>
                                    <a href="http://${EC2_PUBLIC_IP}">
                                        http://${EC2_PUBLIC_IP}
                                    </a>
                                </td>
                            </tr>
                            <tr>
                                <td><b>Build URL</b></td>
                                <td>
                                    <a href="${BUILD_URL}">
                                        ${BUILD_URL}
                                    </a>
                                </td>
                            </tr>
                        </table>

                        <br>
                        <p style="color: #555;">
                            <b>Trivy Security Reports attached hain:</b><br>
                            &nbsp;&nbsp;• fs-scan.txt — File System Scan<br>
                            &nbsp;&nbsp;• backend-image-scan.txt — Backend Image Scan<br>
                            &nbsp;&nbsp;• frontend-image-scan.txt — Frontend Image Scan<br>
                        </p>
                        <p style="color: gray;">
                            Yeh email automatically Jenkins ne bheji hai.
                        </p>

                    </div>
                </body>
                </html>
            """
        )
    }

    // SIRF FAILURE PAR
    failure {
        emailext(
            to:       'biswajitbehera1868@gmail.com',
            subject:  "Build #${BUILD_NUMBER} - ${JOB_NAME} - FAILED",
            mimeType: 'text/html',
            attachmentsPattern: 'reports/trivy/*.txt',
            body:     """
                <html>
                <body style="font-family: Arial; padding: 20px; background-color: #f4f4f4;">

                    <div style="background-color: white; padding: 30px; border-radius: 8px; border-left: 5px solid red;">

                        <h2 style="color: red;">Build Fail Ho Gaya!</h2>

                        <table border="1" cellpadding="10" cellspacing="0" width="100%">
                            <tr style="background-color: #f9f9f9;">
                                <td><b>Project</b></td>
                                <td>${JOB_NAME}</td>
                            </tr>
                            <tr>
                                <td><b>Build Number</b></td>
                                <td>#${BUILD_NUMBER}</td>
                            </tr>
                            <tr style="background-color: #f9f9f9;">
                                <td><b>Status</b></td>
                                <td style="color: red;"><b>FAILED</b></td>
                            </tr>
                            <tr>
                                <td><b>Duration</b></td>
                                <td>${currentBuild.durationString}</td>
                            </tr>
                            <tr style="background-color: #f9f9f9;">
                                <td><b>Failed Stage</b></td>
                                <td>${currentBuild.result}</td>
                            </tr>
                            <tr>
                                <td><b>Console Logs</b></td>
                                <td>
                                    <a href="${BUILD_URL}console">
                                        Yahan Click Karo — Logs Dekho
                                    </a>
                                </td>
                            </tr>
                        </table>

                        <br>
                        <p style="color: #555;">
                            <b>Trivy Security Reports attached hain:</b><br>
                            &nbsp;&nbsp;• fs-scan.txt — File System Scan<br>
                            &nbsp;&nbsp;• backend-image-scan.txt — Backend Image Scan<br>
                            &nbsp;&nbsp;• frontend-image-scan.txt — Frontend Image Scan<br>
                        </p>
                        <p style="color: red;">
                            Console logs dekho aur error fix karo!
                        </p>
                        <p style="color: gray;">
                            Yeh email automatically Jenkins ne bheji hai.
                        </p>

                    </div>
                </body>
                </html>
            """
        )
    }

    // CLEANUP
    cleanup {
        cleanWs(
            cleanWhenSuccess: true,
            cleanWhenFailure: false,
            cleanWhenAborted: true
        )
    }
}
}