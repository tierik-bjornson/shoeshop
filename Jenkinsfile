pipeline {
    agent any
    environment {
        SSH_KEY_CREDENTIALS = credentials('swarm-manager-ssh')
        STACK_NAME = "shoeshop"
        BACKEND_IMAGE = "192.168.2.55:8443/shoeshop/shoeshop-backend"
        FRONTEND_IMAGE = "192.168.2.55:8443/shoeshop/shoeshop-frontend"
        MANAGER_USER = "ubuntu"
        MANAGER_IP = "18.140.218.13"
        GIT_REPO_URL = "https://github.com/tierik-bjornson/shoeshop.git"
        GIT_BRANCH = "main"
        REGISTRY_URL = "192.168.2.55:8443"
    }

    stages {
        stage('Checkout Code') {
            steps {
                git branch: "${GIT_BRANCH}", url: "${GIT_REPO_URL}"
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
                    sh """
                        docker build -t ${BACKEND_IMAGE}:latest .
                        docker tag ${BACKEND_IMAGE}:latest ${BACKEND_IMAGE_TAGGED}
                    """
                }
            }
        }

        stage('Build Frontend Image') {
            steps {
                dir('Frontend') {
                    sh """
                        docker build -t ${FRONTEND_IMAGE}:latest .
                        docker tag ${FRONTEND_IMAGE}:latest ${FRONTEND_IMAGE_TAGGED}
                    """
                }
            }
        }

        stage('Image Scan backend') {
            steps {
                script {
                    docker.image('aquasec/trivy:latest').inside('--dns 8.8.8.8 --entrypoint="" --volume /home/tien/trivy/html.tpl:/html.tpl:ro') {
                        sh """
                            mkdir -p trivy-reports
                            chmod -R 777 trivy-reports

                            cp /html.tpl trivy-reports/html.tpl || { echo "Failed  to copy html.tpl"; exit 1; }

                            if [ ! -s trivy-reports/html.tpl ]; then
                                echo "Error:html.tpl is empty or not found"
                                exit 1
                            fi

                            trivy image --format json --severity HIGH,CRITICAL --output trivy-reports/backend.json $BACKEND_IMAGE || { echo "Failed to scan backend image"; exit 1; }

                            trivy convert --format template --template @trivy-reports/html.tpl --output trivy-reports/backend.html trivy-reports/backend.json || { echo "Failed to convert JSON to HTML"; exit 1; }
                        """

                        archiveArtifacts artifacts: 'trivy-reports/backend.*', fingerprint: true
                    }
                }
            }
        }

        stage('Push Images to Harbor') {
            steps {
                withDockerRegistry(credentialsId: 'harbor-credentials', url: "https://${REGISTRY_URL}") {
                    sh """
                        docker push ${BACKEND_IMAGE}:latest
                        docker push ${BACKEND_IMAGE_TAGGED}
                        docker push ${FRONTEND_IMAGE}:latest
                        docker push ${FRONTEND_IMAGE_TAGGED}
                    """
                }
            }
        }

        
        stage('Pull docker-compose.yml from Git') {
            steps {
                sshagent(['swarm-manager-ssh']) {
                       sh """
                       ssh -o StrictHostKeyChecking=no $MANAGER_USER@$MANAGER_IP '
                       rm -rf /home/$MANAGER_USER/shoeshop
                        git clone -b $GIT_BRANCH $GIT_REPO_URL /home/$MANAGER_USER/shoeshop
                       '
                        """

               }

            }
        }

        stage('Deploy Docker Stack via SSH') {
            steps {
                sshagent(['swarm-manager-ssh']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${MANAGER_USER}@${MANAGER_IP} '
                            export TAG=${IMAGE_TAG}
                            docker stack deploy -c /home/${MANAGER_USER}/shoeshop/docker-compose.yml ${STACK_NAME} --with-registry-auth
                        '
                    """
                }
            }
        }

        // stage('Health check') {
        //     steps {
        //         sshagent(['swarm-manager-ssh']) {
        //             sh '''
        //                 ssh -o StrictHostKeyChecking=no ${MANAGER_USER}@${MANAGER_IP} "
        //                     netstat -tuln | grep 3000 &&
        //                     curl --fail --max-time 100 -v http://${MANAGER_IP}:3000
        //                 "
        //             '''
        //         }
        //     }
        // }
    }
}
