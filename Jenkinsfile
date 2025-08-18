pipeline {
    agent any
    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')  
        SSH_KEY_CREDENTIALS = credentials('swarm-manager-ssh')       
        STACK_NAME = "shoeshop"
        BACKEND_IMAGE = "tien2k3/shoeshop-backend:latest"
        FRONTEND_IMAGE = "tien2k3/shoeshop-frontend:latest"
        MANAGER_USER = "ubuntu"
        MANAGER_IP = "EC2_PUBLIC_IP"
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/tierik-bjornson/shoeshop.git'
            }
        }

        stage('Build Backend Image') {
            steps {
                dir('Backend') {
                    sh "docker build -t $BACKEND_IMAGE ."
                }
            }
        }

        stage('Build Frontend Image') {
            steps {
                dir('Frontend') {
                    sh "docker build -t $FRONTEND_IMAGE ."
                }
            }
        }

        stage('Trivy Scan Backend') {
            steps {
                sh "trivy image --exit-code 1 --severity HIGH,CRITICAL $BACKEND_IMAGE || true"
            }
        }

        stage('Trivy Scan Frontend') {
            steps {
                sh "trivy image --exit-code 1 --severity HIGH,CRITICAL $FRONTEND_IMAGE || true"
            }
        }

        stage('Push Images to DockerHub') {
            steps {
                script {
                    sh """
                    echo $DOCKERHUB_CREDENTIALS_PSW | docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin
                    docker push $BACKEND_IMAGE
                    docker push $FRONTEND_IMAGE
                    """
                }
            }
        }

        stage('Deploy Docker Stack via SSH') {
            steps {
                sshagent(['swarm-manager-ssh']) { 
                    sh """
                    ssh -o StrictHostKeyChecking=no $MANAGER_USER@$MANAGER_IP '
                        docker stack deploy -c /home/ubuntu/shoeshop/docker-compose.yml $STACK_NAME --with-registry-auth
                    '
                    """
                }
            }
        }
    }

    post {
        always {
            sh 'docker logout'
        }
    }
}
