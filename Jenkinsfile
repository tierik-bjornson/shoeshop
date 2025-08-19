pipeline {
    agent any
    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')  
        SSH_KEY_CREDENTIALS = credentials('swarm-manager-ssh')       
        STACK_NAME = "shoeshop"
        BACKEND_IMAGE = "tien2k3/shoeshop-backend:latest"
        FRONTEND_IMAGE = "tien2k3/shoeshop-frontend:latest"
        MANAGER_USER = "ubuntu"
        MANAGER_IP = "18.140.218.13"
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

        stage('Image Scan backend') {
    steps {
        script {
            docker.image('aquasec/trivy:latest').inside('--dns 8.8.8.8 --entrypoint=""') {
                sh """
                # Tạo thư mục lưu report
                mkdir -p trivy-reports

                # Quét container backend và xuất JSON
                trivy image --format json --severity HIGH,CRITICAL --output trivy-reports/backend.json $BACKEND_IMAGE || { echo "Failed to scan backend image"; exit 1; }

                # Tải template HTML về nếu chưa có
                if [ ! -f trivy-reports/html.tpl ]; then
                    curl -sSL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/html.tpl -o trivy-reports/html.tpl || { echo "Failed to download html.tpl"; exit 1; }
                fi

                # Kiểm tra xem file html.tpl có nội dung không
                if [ ! -s trivy-reports/html.tpl ]; then
                    echo "Error: html.tpl is empty or not downloaded correctly"
                    exit 1
                fi

                # Chuyển JSON sang HTML
                trivy convert --format template --template @trivy-reports/html.tpl --output trivy-reports/backend.html trivy-reports/backend.json || { echo "Failed to convert JSON to HTML"; exit 1; }
                """

                // Lưu artifacts để xem trực tiếp trong Jenkins
                archiveArtifacts artifacts: 'trivy-reports/backend.*', fingerprint: true
            }
        }
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

        stage('Copy docker-compose.yml to Manager') {
            steps {
                sshagent(['swarm-manager-ssh']) {
                     sh '''
                     ssh -o StrictHostKeyChecking=no ubuntu@18.140.218.13 "mkdir -p /home/ubuntu/shoeshop"
                     scp -o StrictHostKeyChecking=no docker-compose.yml ubuntu@18.140.218.13:/home/ubuntu/shoeshop/docker-compose.yml
                     '''
                }
            }
        }

        stage('Deploy Docker Stack via SSH') {
            steps {
                sshagent(['swarm-manager-ssh']) {
                    sh """
                    ssh -o StrictHostKeyChecking=no $MANAGER_USER@$MANAGER_IP '
                        docker stack deploy -c /home/$MANAGER_USER/shoeshop/docker-compose.yml $STACK_NAME --with-registry-auth
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
