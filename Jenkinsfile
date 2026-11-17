pipeline {
    agent any

    environment {
        IMAGE_NAME = 'it-service-desk'
        EC2_HOST = '18.189.186.173'
        EC2_USER = 'ubuntu'
        EC2_DIR = '/home/ubuntu/it-service-desk'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git --version'
            }
        }

        stage('Automated Tests') {
            steps {
                sh '''
                    python3 -m venv .venv
                    . .venv/bin/activate
                    pip install --no-cache-dir -r requirements.txt
                    python -m unittest discover -s tests -v
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build --pull -t ${IMAGE_NAME}:${BUILD_NUMBER} .
                '''
            }
        }

        stage('Docker Smoke Test') {
            steps {
                sh '''
                    docker rm -f it-service-desk-smoke >/dev/null 2>&1 || true

                    docker run -d \
                      --name it-service-desk-smoke \
                      -p 5002:5000 \
                      ${IMAGE_NAME}:${BUILD_NUMBER}

                    trap 'docker rm -f it-service-desk-smoke >/dev/null 2>&1 || true' EXIT

                    for i in $(seq 1 20); do
                        if curl -fsS http://127.0.0.1:5002/health; then
                            exit 0
                        fi
                        sleep 2
                    done

                    echo "Smoke test failed"
                    exit 1
                '''
            }
        }

        stage('Deploy to AWS EC2') {
            steps {
                sshagent(credentials: ['ec2-ssh-key']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=accept-new \
                            -o ConnectTimeout=15 \
                            ${EC2_USER}@${EC2_HOST} \
                            'cd /home/ubuntu/it-service-desk &&
                             git fetch origin &&
                             git reset --hard origin/main &&
                             sudo docker compose up -d --build &&
                             for i in $(seq 1 20); do
                               if curl -fsS http://localhost:5001/health\; then
                                 exit 0
                               fi
                               sleep 2
                             done &&
                             echo "EC2 health check failed" &&
                             exit 1'
                    '''
                }
            }
        }
    }

    post {
        always {
            sh 'docker logs it-service-desk-smoke --tail 30 2>/dev/null || true'
        }

        success {
            echo 'SUCCESS: IT Service Desk deployed to AWS EC2.'
        }

        failure {
            echo 'FAILED: Check the Jenkins Console Output for details.'
        }
    }
}
