pipeline {
    agent any

    environment {
        IMAGE_NAME = 'it-service-desk'
        EC2_HOST = '18.189.186.173'
        EC2_USER = 'ubuntu'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
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
                sh 'docker build --pull -t ${IMAGE_NAME}:${BUILD_NUMBER} .'
            }
        }

        stage('Docker Smoke Test') {
            steps {
                sh '''
                    docker rm -f it-service-desk-smoke || true

                    docker run -d \
                        --name it-service-desk-smoke \
                        -p 5002:5000 \
                        ${IMAGE_NAME}:${BUILD_NUMBER}

                    for i in $(seq 1 20); do
                        if curl -fsS http://127.0.0.1:5002/health; then
                            docker rm -f it-service-desk-smoke
                            exit 0
                        fi
                        sleep 2
                    done

                    docker logs it-service-desk-smoke || true
                    docker rm -f it-service-desk-smoke || true
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
                            'cd ~/it-service-desk &&
                             git fetch origin &&
                             git reset --hard origin/main &&
                             sudo docker compose up -d --build &&
                             sleep 5 &&
                             curl -fsS http://localhost:5001/health'
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: IT Service Desk deployed to AWS EC2.'
        }

        failure {
            echo 'FAILED: Check Console Output for the failed stage.'
        }
    }
}
