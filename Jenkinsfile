pipeline {
    agent any
    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')  
        SSH_KEY_CREDENTIALS = credentials('swarm-manager-ssh')       
        STACK_NAME = "shoeshop"
        BACKEND_IMAGE = "tien2k3/shoeshop-backend"
        FRONTEND_IMAGE = "tien2k3/shoeshop-frontend"
        MANAGER_USER = "ubuntu"
        MANAGER_IP = "18.140.218.13"
        GIT_REPO_URL = "https://github.com/tierik-bjornson/shoeshop.git"
        GIT_BRANCH = "main"
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: 'main', url: 'https://github.com/tierik-bjornson/shoeshop.git'
            }
        }

        stage('Set Build Number Tag') {
            steps {
                script {
                    env.IMAGE_TAG = "${BUILD_NUMBER}"
                    env.BACKEND_IMAGE_TAGGED = "${BACKEND_IMAGE}:${IMAGE_TAG}"
                    env.FRONTEND_IMAGE_TAGGED = "${FRONTEND_IMAGE}:${IMAGE_TAG}"
                }
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

        stage('Image Scan backend') {
    steps {
        script {
            docker.image('aquasec/trivy:latest').inside('--dns 8.8.8.8 --entrypoint="" --volume /usr/bin/html.tpl:/html.tpl:ro') {
                sh """
           
                mkdir -p trivy-reports
                chmod -R 777 trivy-reports

                # Copy html.tpl từ volume vào trivy-reports
                cp /html.tpl trivy-reports/html.tpl || { echo "Failed to copy html.tpl"; exit 1; }

                # Kiểm tra xem file html.tpl có nội dung không
                if [ ! -s trivy-reports/html.tpl ]; then
                    echo "Error: html.tpl is empty or not found"
                    exit 1
                fi

                # Quét container backend và xuất JSON
                trivy image --format json --severity HIGH,CRITICAL --output trivy-reports/backend.json $BACKEND_IMAGE || { echo "Failed to scan backend image"; exit 1; }

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
                    docker push ${BACKEND_IMAGE}:latest
                    docker push ${BACKEND_IMAGE_TAGGED}
                    docker push ${FRONTEND_IMAGE}:latest
                    docker push ${FRONTEND_IMAGE_TAGGED}
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
