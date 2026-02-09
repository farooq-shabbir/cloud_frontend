pipeline {
    agent any
    
    environment {
        // Credentials should be configured in Jenkins Credentials Manager
        DOCKER_CREDS = credentials('docker-hub-credentials') // Username/Password
        EC2_SSH_KEY  = credentials('ec2-ssh-key')            // SSH Private Key (Secret file or text)
        EC2_HOST     = credentials('ec2-server-ip')          // Secret Text credential containing IP
        EC2_USER     = 'ubuntu'
        DOCKER_REPO  = credentials('docker-repo')            // Secret Text credential containing 'username/repo'
        IMAGE_TAG    = "${DOCKER_REPO}:${BUILD_NUMBER}"
        LATEST_TAG   = "${DOCKER_REPO}:latest"
        CONTAINER_NAME = "myapp-jenkins"
        BACKUP_NAME    = "myapp-jenkins-backup"
        PORT           = "3038"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install & Lint') {
            agent {
                docker {
                    image 'node:20'
                    reuseNode true
                }
            }
            steps {
                dir('frontend') {
                    sh 'npm ci'
                    sh 'npm run lint'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    dir('frontend') {
                        sh "docker build -f Dockerfile.jenkins -t ${IMAGE_TAG} -t ${LATEST_TAG} ."
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'docker-hub-credentials', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                        sh "docker push ${IMAGE_TAG}"
                        sh "docker push ${LATEST_TAG}"
                    }
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                script {
                    sshagent(['ec2-ssh-key']) { // Requires SSH Agent Plugin in Jenkins
                        sh """
                            ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} '
                                set -e
                                
                                # --- Functions ---
                                retry() {
                                    local n=1
                                    local max=3
                                    local delay=5
                                    while true; do
                                        "\$@" && break || {
                                            if [[ \$n -lt \$max ]]; then
                                                ((n++))
                                                echo "Command failed. Attempt \$n/\$max:"
                                                sleep \$delay;
                                            else
                                                echo "The command has failed after \$n attempts."
                                                return 1
                                            fi
                                        }
                                    done
                                }

                                check_health() {
                                    local url="http://localhost:${PORT}"
                                    local max_attempts=12
                                    local attempt=1
                                    
                                    echo "Checking health at \$url..."
                                    while [ \$attempt -le \$max_attempts ]; do
                                        if curl -s -f "\$url" > /dev/null; then
                                            echo "Health check passed!"
                                            return 0
                                        fi
                                        echo "Health check attempt \$attempt/\$max_attempts failed. Retrying in 5s..."
                                        sleep 5
                                        ((attempt++))
                                    done
                                    return 1
                                }

                                # 1. Install Docker if missing (Simplified check)
                                if ! command -v docker &> /dev/null; then
                                    echo "Docker not found. Installing..."
                                    # (Insert installation commands here if needed, assuming already installed by previous pipelines)
                                    sudo apt-get update && sudo apt-get install -y docker.io
                                    sudo usermod -aG docker \$USER
                                fi

                                # 2. Login to Docker Hub (Pass credentials via env vars safely if possible, or assume public pull if public)
                                # Note: In a real Jenkins env, we might need to pass the token explicitly.
                                # For now, we assume we can pull if public, or we need to pass creds.
                                # Let us skip login if repo is public, otherwise we need to pipe DOCKER_PASS.
                                
                                # 3. Pull Image
                                echo "Pulling image: ${IMAGE_TAG}"
                                retry sudo docker pull ${IMAGE_TAG}

                                # 4. Prepare Rollback
                                if sudo docker ps -a --format "{{.Names}}" | grep -q "^${CONTAINER_NAME}\$"; then
                                    echo "Existing container found. Backing up..."
                                    sudo docker stop ${CONTAINER_NAME} || true
                                    sudo docker rename ${CONTAINER_NAME} ${BACKUP_NAME}
                                    sudo docker stop ${BACKUP_NAME} || true
                                fi

                                # 5. Run New Container
                                echo "Starting new container on port ${PORT}..."
                                sudo fuser -k ${PORT}/tcp || true
                                sudo docker run -d \\
                                    --name ${CONTAINER_NAME} \\
                                    --restart always \\
                                    -p ${PORT}:${PORT} \\
                                    ${IMAGE_TAG}

                                # 6. Health Check & Rollback
                                if check_health; then
                                    echo "Deployment successful."
                                    if sudo docker ps -a --format "{{.Names}}" | grep -q "^${BACKUP_NAME}\$"; then
                                        sudo docker rm -f ${BACKUP_NAME}
                                    fi
                                else
                                    echo "Health check FAILED. Rolling back..."
                                    sudo docker rm -f ${CONTAINER_NAME}
                                    
                                    if sudo docker ps -a --format "{{.Names}}" | grep -q "^${BACKUP_NAME}\$"; then
                                        sudo docker rename ${BACKUP_NAME} ${CONTAINER_NAME}
                                        sudo docker start ${CONTAINER_NAME}
                                        echo "Rollback completed."
                                    else
                                        echo "No backup found. Service is down."
                                    fi
                                    exit 1
                                fi

                                # 7. Cleanup
                                sudo docker image prune -f
                            '
                        """
                    }
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
    }
}
