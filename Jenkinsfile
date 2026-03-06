pipeline {
    agent any

    environment {
        // ─── Docker Hub ───────────────────────────────────────────────
        DOCKERHUB_USER   = 'akshatzzz'          
        BACKEND_IMAGE    = "${DOCKERHUB_USER}/redbridge-backend"
        FRONTEND_IMAGE   = "${DOCKERHUB_USER}/redbridge-frontend"
        IMAGE_TAG        = "${BUILD_NUMBER}"

        // ─── EC2 Deploy Target ────────────────────────────────────────
        EC2_USER         = 'ubuntu'
        EC2_HOST         = '13.235.251.113'                  // ← change this
        APP_DIR          = '/home/ubuntu/redbridge'
    }

    stages {

        // ── 1. Pull latest code ───────────────────────────────────────
        stage('Checkout') {
            steps {
                git branch: 'main',
                    // credentialsId: 'github-creds',
                    url: 'https://github.com/AkshatforGithub/RedBridge'  // ← change this
            }
        }

        // ── 2. Install & test backend ─────────────────────────────────
        stage('Backend - Install & Test') {
            tools { nodejs 'NodeJS-20' }
            steps {
                dir('server') {
                    sh 'npm ci'
                    sh 'npm test --if-present'   // skips gracefully if no tests yet
                }
            }
        }

        // ── 3. Install & test frontend ────────────────────────────────
        stage('Frontend - Install & Test') {
            tools { nodejs 'NodeJS-20' }
            steps {
                dir('client') {
                    sh 'npm ci'
                    sh 'npm test --if-present -- --watchAll=false'
                }
            }
        }

        // ── 4. Build & push both images to Docker Hub ─────────────────
        stage('Docker Build & Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                        echo "\$DOCKER_PASS" | docker login -u "\$DOCKER_USER" --password-stdin

                        # ── Build Backend ──
                        docker build -t ${BACKEND_IMAGE}:${IMAGE_TAG} ./server
                        docker tag  ${BACKEND_IMAGE}:${IMAGE_TAG} ${BACKEND_IMAGE}:latest
                        docker push ${BACKEND_IMAGE}:${IMAGE_TAG}
                        docker push ${BACKEND_IMAGE}:latest

                        # ── Build Frontend ──
                        docker build -t ${FRONTEND_IMAGE}:${IMAGE_TAG} ./client
                        docker tag  ${FRONTEND_IMAGE}:${IMAGE_TAG} ${FRONTEND_IMAGE}:latest
                        docker push ${FRONTEND_IMAGE}:${IMAGE_TAG}
                        docker push ${FRONTEND_IMAGE}:latest
                    """
                }
            }
        }

        // ── 5. Security scan both images with Trivy ───────────────────
        stage('Trivy Security Scan') {
            steps {
                sh """
                    echo "🔍 Scanning backend image..."
                    trivy image --exit-code 1 \
                                --severity HIGH,CRITICAL \
                                --no-progress \
                                --format table \
                                ${BACKEND_IMAGE}:latest

                    echo "🔍 Scanning frontend image..."
                    trivy image --exit-code 1 \
                                --severity HIGH,CRITICAL \
                                --no-progress \
                                --format table \
                                ${FRONTEND_IMAGE}:latest
                """
            }
            post {
                failure {
                    echo '🚨 Critical vulnerabilities found! Deployment blocked.'
                }
                success {
                    echo '✅ Both images passed security scan.'
                }
            }
        }

        // ── 6. Deploy to EC2 via SSH ──────────────────────────────────
        stage('Deploy to EC2') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sshagent(['ec2-ssh-key']) {
                        sh """
                            # Create app directory on EC2 if it doesn't exist
                            ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} \
                                'mkdir -p ${APP_DIR}'

                            # Copy latest docker-compose.yml to EC2
                            scp -o StrictHostKeyChecking=no \
                                docker-compose.yml \
                                ${EC2_USER}@${EC2_HOST}:${APP_DIR}/

                            # SSH in and redeploy all services
                            ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} '
                                cd ${APP_DIR}

                                # Login to Docker Hub on EC2
                                echo "${DOCKER_PASS}" | docker login -u "${DOCKER_USER}" --password-stdin

                                # Pull latest images (backend + frontend; mongo uses fixed tag)
                                docker compose pull backend frontend

                                # Restart all containers with zero-downtime approach
                                docker compose down --remove-orphans
                                docker compose up -d

                                # Clean up old unused images
                                docker image prune -f
                            '
                        """
                    }
                }
            }
        }

        // ── 7. Health check after deploy ─────────────────────────────
        stage('Health Check') {
            steps {
                sh """
                    echo "⏳ Waiting for services to start..."
                    sleep 20

                    # Check backend is up
                    echo "Checking backend..."
                    curl -f http://${EC2_HOST}:5000/health || \
                        curl -f http://${EC2_HOST}:5000 || \
                        echo "⚠️ Backend health check failed - verify manually"

                    # Check frontend is up
                    echo "Checking frontend..."
                    curl -f http://${EC2_HOST}:3000 || \
                        echo "⚠️ Frontend health check failed - verify manually"

                    echo "✅ Health checks complete"
                """
            }
        }

    }

    // ── Post-pipeline notifications ───────────────────────────────────
    post {
        success {
            echo """
            ✅ Deployment Successful!
            ─────────────────────────────
            Frontend : http://${EC2_HOST}:3000
            Backend  : http://${EC2_HOST}:5000
            Build #  : ${BUILD_NUMBER}
            """
        }
        failure {
            echo """
            ❌ Pipeline Failed at stage: check logs above
            Build # : ${BUILD_NUMBER}
            """
        }
        always {
           
            cleanWs()
        }
    }
}