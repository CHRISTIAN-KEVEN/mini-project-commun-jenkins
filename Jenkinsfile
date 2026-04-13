pipeline {
    agent none
    environment {
        DOCKERHUB_AUTH = credentials('DOCKER_HUB_ID')
        DOCKERHUB_USERNAME = 'kreys326'
        IMAGE_NAME = 'static-website'
        IMAGE_TAG = 'v1'
        PORT_EXPOSED = '80'
        PROD_ID = "51.20.106.82"
        STAGING_IP = "13.60.5.88"

    }

    stages {

        stage('Build') {
            agent any
            steps {
                script {
                    sh 'docker build -t ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${IMAGE_TAG} .'
                }
            }
        }
        stage('Run and test') {
            agent any
            steps {
                script {
                    sh '''
                          docker rm -f ${IMAGE_NAME} || echo 'Container does not exist'
                          docker run --name ${IMAGE_NAME} -d -p ${PORT_EXPOSED}:80 ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${IMAGE_TAG}
                          sleep 15
                          curl http://172.17.0.1:${PORT_EXPOSED} | grep -q "A fully responsive site template designed by"
                    '''
                }
            }
        }
        stage('Clean Container and Push Image') {
            agent any
            steps {
                script {
                    sh '''
                        docker stop ${IMAGE_NAME}
                        docker rm ${IMAGE_NAME}
                        docker login -u ${DOCKERHUB_AUTH_USR} -p ${DOCKERHUB_AUTH_PSW}
                        docker push ${DOCKERHUB_AUTH_USR}/${IMAGE_NAME}:${IMAGE_TAG}
                    '''
                }
            }
        }
        stage('Deploy to stage') {
            // environment {
            //     STAGING_IP = "13.60.5.88"
            // }
            agent any
            steps {
                sshagent(['SSH_AUTH_KEY']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no ubuntu@$STAGING_IP "
                            docker rm -f $IMAGE_NAME || echo 'All deleted' && \
                            docker pull $DOCKERHUB_USERNAME/$IMAGE_NAME:$IMAGE_TAG || echo 'Image Download successfully' && \
                            docker run --rm -dp $PORT_EXPOSED:80 --name $IMAGE_NAME $DOCKERHUB_USERNAME/$IMAGE_NAME:$IMAGE_TAG && \
                            sleep 15 && \
                            curl -I http://localhost:$PORT_EXPOSED && \
                            echo 'Deployment successful'
                        "
                    '''
                }
            }
        }
        stage('Deploy to prod') {
            when {
                expression { env.BRANCH_NAME == 'origin/main' }
            }
            // environment {
            //     PROD_ID = "51.20.106.82"
            // }
            agent any
            steps {
                sshagent(['SSH_AUTH_KEY']) {
                    sh '''
                        mkdir -p ~/.ssh
                        chmod 0700 ~/.ssh
                        ssh-keyscan -t ed25519 -H $PROD_ID >> ~/.ssh/known_hosts
                        ssh ubuntu@$PROD_ID "
                            docker rm -f ${IMAGE_NAME} || echo 'Already deleted' && \
                            docker pull ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${IMAGE_TAG} || echo 'Image Download successfully' && \
                            docker run --rm -d -p $PORT_EXPOSED:80 --name $IMAGE_NAME ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${IMAGE_TAG} && \
                            sleep 15 && \
                            curl -I http://localhost:$PORT_EXPOSED && \
                            echo 'Deployment successful'
                        "
                        
                    '''
                }
            }
            

        }
    }

    post {
       success {
         slackSend (color: '#00FF00', message: "AJEITOH - SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL}) - PROD URL => http://${PROD_ID} , STAGING URL => http://${STAGING_IP}")
         }
      failure {
            slackSend (color: '#FF0000', message: "AJEITOH - FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})")
          }   
    }  
}
