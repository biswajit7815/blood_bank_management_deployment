pipeline {
    agent any

    tools {
        nodejs 'node18'
    }

    // all environment variable i used this so entire pipeline me when need these i only used ok..

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

        // stage 1: clean and checkout

        stage('cleanup & checkout') {
            steps {
                cleanWs()
                checkout scm
                echo "code checkout complete - Build #${BUiLD_NUMBER}"
            }
        }

        // stage: 2 install all dependncies with parallel
        // why i used this because fast install even time consume nehi lega

        stage('install dependencies') {
            parallel {

                stage('Backend install') {
                    steps {
                        dir('backend') {
                            sh 'npm install'
                        }
                    }
                }

                stage('frontend install') {
                    steps {
                        dir('frontend') {
                            sh 'npm install'
                        }
                    }
                }
            }
        }

        // build the docker images

        stage('Build Docker images') {
            steps {

                // build the backend image....

                echo "build the backend image...."

                sh """
                    docker build \
                        -t ${DOCKERHUB_USERNAME}/${BACKEND_IMAGE}:${IMAGE_TAG} \
                        -t ${DOCKERHUB_USERNAME}/${BACKEND_IMAGE}:latest \
                        -f backend/Dockerfile \
                        ./backend
                """

                // build the fronted image.....

                echo "build frontend image..."

                sh """
                    docker build \
                        --build-arg VITE_API_PATH=http://${EC2_PUBLIC_IP}:${BACKEND_PORT} \
                        -t ${DOCKERHUB_USERNAME}/${FRONTEND_IMAGE}:${IMAGE_TAG} \
                        -t ${DOCKERHUB_USERNAME}/${FRONTEND_IMAGE}:latest \
                        -f frontend/Dockerfile
                        ./frontend
                """
            }
        }

        stage('push to docker hub') {
            steps {
                script {

                    // Jenkins ke stored secret credentials use karta hai.

                    withCredentials([
                        usernamePassword(
                            credentialsId: 'docker-hub-creds',
                            passwordVariable: 'DOCKER_PASS',
                            usernameVariable: 'DOCKER_USER'
                        )
                    ]) {

                        sh "echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin"

                        sh "docker push ${DOCKERHUB_USERNAME}/${BACKEND_IMAGE}:${IMAGE_TAG}"
                        sh "docker push ${DOCKERHUB_USERNAME}/${BACKEND_IMAGE}:latest"

                        sh "docker push ${DOCKERHUB_USERNAME}/${FRONTEND_IMAGE}:${IMAGE_TAG}"
                        sh "docker push ${DOCKERHUB_USERNAME}/${FRONTEND_IMAGE}:latest"

                        sh "docker logout"
                    }
                }
            }
        }

        // deploy the application

        stage('deploy') {
            steps {
                script {

                    withCredentials([
                        string(credentialsId: 'MONGO_URI', variable: 'MONGO_URI'),
                        string(credentialsId: 'JWT_SECRET', variable: 'JWT_SECRET'),
                    ]) {

                        echo "stopping old container........"

                        sh "docker stop ${BACKEND_CONTAINER} || true"
                        sh "docker stop ${FRONTEND_CONTAINER} || true"
                        sh "docker rm ${BACKEND_CONTAINER} || ture"
                        sh "docker rm ${FRONTEND_CONTAINER} || ture"

                        echo "creating docker network...."

                        sh "docker network create blood-network || true"

                        echo "starting Backend container....."

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

                        echo "staring frontend container......."

                        sh """
                            docker run -d \
                                --name ${FRONTEND_CONTAINER} \
                                --network blood-network \
                                --restart unless-stopped \
                                -p 80:80 \
                                ${DOCKERHUB_USERNAME}/${FRONTEND_IMAGE}:${IMAGE_TAG}
                        """

                        echo "Waiting for containers to start..."

                        sh "sleep 15"

                        echo "Container Status"

                        sh "docker ps --filter 'name=${BACKEND_CONTAINER}'"
                        sh "docker ps --filter 'name=${FRONTEND_CONTAINER}'"

                        echo "Backend Health Check"

                        sh "curl -sf http://localhost:${BACKEND_PORT}/api/health && echo 'Backend Healthy' || echo 'Backend Health Failed'"

                        echo "Frontend Health Check"

                        sh "curl -sf http://localhost && echo 'Frontend Healthy' || echo 'Frontend Health Failed'"

                        echo "Application Live : http://${EC2_PUBLIC_IP}"
                    }
                }
            }
        }

        // cleanup the old image

        stage('cleanup old images') {
            steps {

                echo "Cleaning dangling images..."

                sh "docker image prune -f"

                echo "Removing the old backend image......"

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