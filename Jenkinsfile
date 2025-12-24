pipeline {
    agent any

    environment {
        IMAGE_NAME     = 'moiz3388/ace13:latest'
        CONTAINER_NAME = 'aceserver'
        BROKER_NAME    = 'PROD'
        SERVER_NAME    = 'prodserver'
        ACE_PROFILE    = '/opt/ibm/ace-13/server/bin/mqsiprofile'
        BAR_FILE       = 'BVSRegFix2.bar'
    }

    stages {

        stage('Clean & Start ACE Container') {
            steps {
                sh '''
                    echo "Cleaning any existing ACE container..."
                    docker rm -f ${CONTAINER_NAME} || true

                    echo "Starting fresh ACE container..."
                    docker run -d --name ${CONTAINER_NAME} \
                        -e LICENSE=accept \
                        ${IMAGE_NAME} tail -f /dev/null

                    echo "Waiting for container to be ready..."
                    sleep 10
                '''
            }
        }

        stage('Configure Broker & Server') {
            steps {
                sh '''
                    docker exec ${CONTAINER_NAME} bash -l -c '
                        set -e
                        . ${ACE_PROFILE}

                        echo "Checking broker..."
                        if ! mqsilist | grep -q ${BROKER_NAME}; then
                            echo "Creating broker ${BROKER_NAME}"
                            mqsicreatebroker ${BROKER_NAME}
                        fi

                        if mqsilist | grep -q "${BROKER_NAME}.*running"; then
                            echo "Stopping broker before server creation"
                            mqsistop ${BROKER_NAME}
                        fi

                        if ! mqsilist ${BROKER_NAME} | grep -q ${SERVER_NAME}; then
                            echo "Creating server ${SERVER_NAME}"
                            mqsicreateexecutiongroup ${BROKER_NAME} -e ${SERVER_NAME}
                        fi

                        echo "Starting broker ${BROKER_NAME}"
                        mqsistart ${BROKER_NAME}

                        echo "Final status:"
                        mqsilist ${BROKER_NAME}
                    '
                '''
            }
        }

        stage('Deploy BAR File') {
            steps {
                sh '''
                    echo "Copying BAR file into container..."
                    docker cp ${BAR_FILE} ${CONTAINER_NAME}:/tmp/${BAR_FILE}

                    docker exec ${CONTAINER_NAME} bash -l -c '
                        set -e
                        . ${ACE_PROFILE}

                        echo "Deploying BAR file..."
                        mqsideploy ${BROKER_NAME} \
                            -e ${SERVER_NAME} \
                            -a /tmp/${BAR_FILE} \
                            -w 120

                        echo "Deployment complete."
                        mqsilist ${BROKER_NAME} -e ${SERVER_NAME}
                    '
                '''
            }
        }
    }
}
