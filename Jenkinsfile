pipeline {
    agent any

    environment {
        REGISTRY       = 'docker.io'
        IMAGE_NAME     = 'rohitg0255/project02-kbd'      // no docker.io/ prefix
        CONTAINER_NAME = 'project02-kbd'
        APP_PORT       = '8080'
        HOST_PORT      = '80'
        HEALTH_URL     = "http://localhost:${HOST_PORT}/"
        REGISTRY_CREDS = 'dockerhub_creds'
        EC2_SSH_CREDS  = 'ec2-ssh-key'
        EC2_HOST       = 'ec2-user@13.204.46.212'          // <-- change to your EC2
        IMAGE_TAG      = "${env.BUILD_NUMBER}-${env.GIT_COMMIT.take(7)}"
    }

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Build + Tag') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} -t ${IMAGE_NAME}:latest ."
            }
        }

        stage('Login + Push') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: env.REGISTRY_CREDS,
                    usernameVariable: 'REG_USER',
                    passwordVariable: 'REG_PASS'
                )]) {
                    sh '''
                        echo "$REG_PASS" | docker login -u "$REG_USER" --password-stdin
                        docker push "$IMAGE_NAME:$IMAGE_TAG"
                        docker push "$IMAGE_NAME:latest"
                    '''
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: env.REGISTRY_CREDS,
                    usernameVariable: 'REG_USER',
                    passwordVariable: 'REG_PASS'
                )]) {
                    sshagent([env.EC2_SSH_CREDS]) {
                        sh """
                            ssh -o StrictHostKeyChecking=no ${EC2_HOST} '
                                echo "$REG_PASS" | docker login -u "$REG_USER" --password-stdin
                                PREV=\$(docker inspect --format="{{.Config.Image}}" ${CONTAINER_NAME} 2>/dev/null || true)
                                echo "\$PREV" > /tmp/prev_image_${CONTAINER_NAME}
                                docker pull ${IMAGE_NAME}:${IMAGE_TAG}
                                docker rm -f ${CONTAINER_NAME} 2>/dev/null || true
                                docker run -d --name ${CONTAINER_NAME} --restart unless-stopped \
                                  -p ${HOST_PORT}:${APP_PORT} ${IMAGE_NAME}:${IMAGE_TAG}
                            '
                        """
                    }
                }
            }
        }

        stage('Health Check') {
            steps {
                sshagent([env.EC2_SSH_CREDS]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${EC2_HOST} '
                            for i in \$(seq 1 20); do
                                wget -q --spider ${HEALTH_URL} && exit 0
                                sleep 3
                            done
                            exit 1
                        '
                    """
                }
            }
        }
    }

    post {
        success { echo "Deploy OK: ${IMAGE_NAME}:${IMAGE_TAG}" }
        failure {
            echo "FAILED — rolling back"
            sshagent([env.EC2_SSH_CREDS]) {
                sh """
                    ssh -o StrictHostKeyChecking=no ${EC2_HOST} '
                        PREV=\$(cat /tmp/prev_image_${CONTAINER_NAME} 2>/dev/null)
                        if [ -n "\$PREV" ] && [ "\$PREV" != "null" ]; then
                            docker rm -f ${CONTAINER_NAME} 2>/dev/null || true
                            docker run -d --name ${CONTAINER_NAME} --restart unless-stopped \
                              -p ${HOST_PORT}:${APP_PORT} \$PREV
                        fi
                    '
                """
            }
        }
        always { sh 'docker logout || true' }
    }


}

