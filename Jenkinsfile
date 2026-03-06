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
        EC2_HOST         = '13.235.251.113'
        APP_DIR          = '/home/ubuntu/redbridge'
    }

    stages {

        // ── 1. Pull latest code ───────────────────────────────────────
        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/AkshatforGithub/RedBridge'
            }
        }

        // ── 2. Install & test (backend + frontend in parallel) ────────
        stage('Install & Test') {
            parallel {
                stage('Backend') {
                    tools { nodejs 'NodeJS-18' }
                    steps {
                        // Spin up a disposable MongoDB on a dynamic port to avoid conflicts
                        sh "docker run -d --name mongo-test-${BUILD_NUMBER} -p \$((27017 + ${BUILD_NUMBER} % 100)):27017 mongo:6"
                        dir('server') {
                            sh 'npm ci'
                            sh 'npm test'
                        }
                    }
                    post {
                        always {
                            sh "docker rm -f mongo-test-${BUILD_NUMBER} || true"
                        }
                    }
                }
                stage('Frontend') {
                    tools { nodejs 'NodeJS-18' }
                    steps {
                        dir('client') {
                            sh 'npm ci'
                            sh 'npm run build'   // no test script; validate production build
                        }
                    }
                }
            }
        }

        // ── 3. Build Docker images ────────────────────────────────────
        stage('Docker Build') {
            steps {
                sh """
                    docker build -t ${BACKEND_IMAGE}:${IMAGE_TAG}  ./server
                    docker tag      ${BACKEND_IMAGE}:${IMAGE_TAG}  ${BACKEND_IMAGE}:latest

                    docker build -t ${FRONTEND_IMAGE}:${IMAGE_TAG} ./client
                    docker tag      ${FRONTEND_IMAGE}:${IMAGE_TAG} ${FRONTEND_IMAGE}:latest
                """
            }
        }

        // ── 4. Security scan BEFORE pushing to registry ───────────────
        stage('Trivy Security Scan') {
            steps {
                sh """
                    echo "🔍 Scanning backend image..."
                    trivy image --exit-code 1 \
                                --severity CRITICAL \
                                --no-progress \
                                --format table \
                                ${BACKEND_IMAGE}:${IMAGE_TAG}

                    echo "🔍 Scanning frontend image..."
                    trivy image --exit-code 1 \
                                --severity CRITICAL \
                                --no-progress \
                                --format table \
                                ${FRONTEND_IMAGE}:${IMAGE_TAG}
                """
            }
            post {
                failure {
                    echo '🚨 Critical vulnerabilities found — push & deploy blocked.'
                }
                success {
                    echo '✅ Both images passed security scan.'
                }
            }
        }

        // ── 5. Push verified images to Docker Hub ─────────────────────
        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

                        docker push $BACKEND_IMAGE:$IMAGE_TAG
                        docker push $BACKEND_IMAGE:latest

                        docker push $FRONTEND_IMAGE:$IMAGE_TAG
                        docker push $FRONTEND_IMAGE:latest
                    '''
                }
            }
        }

        // ── 6. Deploy to EC2 via SSH ──────────────────────────────────
        stage('Deploy to EC2') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    ),
                    file(credentialsId: 'redbridge-env', variable: 'ENV_FILE')
                ]) {
                    sshagent(['ec2-ssh-key']) {
                        sh '''
                            # Create app directory on EC2
                            ssh -o StrictHostKeyChecking=no $EC2_USER@$EC2_HOST \
                                "mkdir -p $APP_DIR"

                            # Copy docker-compose.yml and .env secrets to EC2
                            scp -o StrictHostKeyChecking=no \
                                docker-compose.yml \
                                $EC2_USER@$EC2_HOST:$APP_DIR/

                            scp -o StrictHostKeyChecking=no \
                                "$ENV_FILE" \
                                $EC2_USER@$EC2_HOST:$APP_DIR/.env

                            # Login to Docker Hub on EC2 (password piped via stdin)
                            echo "$DOCKER_PASS" | ssh -o StrictHostKeyChecking=no \
                                $EC2_USER@$EC2_HOST \
                                "docker login -u $DOCKER_USER --password-stdin"

                            # Pull latest images and restart services
                            ssh -o StrictHostKeyChecking=no $EC2_USER@$EC2_HOST "
                                cd $APP_DIR
                                docker compose pull backend frontend
                                docker compose up -d --remove-orphans
                                docker image prune -f
                            "
                        '''
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

                    FAILED=0

                    echo "Checking backend..."
                    curl -sf --retry 3 --retry-delay 5 http://${EC2_HOST}:5000/health || {
                        echo "⚠️ Backend health check failed"
                        FAILED=1
                    }

                    echo "Checking frontend..."
                    curl -sf --retry 3 --retry-delay 5 http://${EC2_HOST}:3000 || {
                        echo "⚠️ Frontend health check failed"
                        FAILED=1
                    }

                    if [ \$FAILED -eq 1 ]; then
                        echo "❌ One or more health checks failed!"
                        exit 1
                    fi

                    echo "✅ All health checks passed"
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
            ❌ Pipeline Failed — check logs above
            Build # : ${BUILD_NUMBER}
            """
        }
        always {
            cleanWs()
        }
    }
}
