pipeline {
    agent any

    environment {
        IMAGE_NAME = "moiz3388/ace13"
        CONTAINER_NAME = "aceserver"
        BROKER_NAME = "PROD"
        SERVER_NAME = "prodserver"
    }

    stages {

        stage('Check & Run ACE Container') {
            steps {
                sh '''
                set -e

                if [ "$(docker ps -aq -f name=${CONTAINER_NAME})" ]; then
                    echo "Container exists. Restarting..."
                    docker restart ${CONTAINER_NAME}
                else
                    echo "Container does not exist. Creating..."
                    docker run -d \
                      --name ${CONTAINER_NAME} \
                      -p 7600:7600 \
                      -p 7800:7800 \
                      -p 7843:7843 \
                      -e LICENSE=accept \
                      ${IMAGE_NAME}
                fi
                '''
            }
        }

        stage('Create Broker & Server (if not exists)') {
            steps {
               sh """
                docker exec ${CONTAINER_NAME} bash -c '
                    # Load ACE environment
                    . /opt/ibm/ace-13/server/bin/mqsiprofile

                    # Create broker if it does not exist
                    if ! mqsilist | grep -q "${BROKER_NAME}"; then
                        echo "Creating broker ${BROKER_NAME}"
                        mqsicreatebroker ${BROKER_NAME} --enable-mq
                    else
                        echo "Broker already exists"
                    fi

                    # Create execution group if it does not exist
                    if ! mqsilist ${BROKER_NAME} | grep -q "${SERVER_NAME}"; then
                        echo "Creating server ${SERVER_NAME}"
                        mqsicreateexecutiongroup ${BROKER_NAME} -e ${SERVER_NAME}
                    else
                        echo "Server already exists"
                    fi

                    # Start broker
                    mqsistart ${BROKER_NAME}
                '
                """


            }
        }

        stage('Ensure Broker & Server Running') {
            steps {
                sh '''
                docker exec ${CONTAINER_NAME} bash -c '
                mqsistart ${BROKER_NAME}
                '
                '''
            }
        }
    }

    post {
        always {
            echo "ACE Pipeline completed"
        }
    }
}
