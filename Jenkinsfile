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
                echo "Cleaning old container (if any)..."
                docker rm -f ${CONTAINER_NAME} || true

                echo "Starting ACE container..."
                docker run -d --name ${CONTAINER_NAME} \
                    -e LICENSE=accept \
                    ${IMAGE_NAME}

                echo "Waiting for ACE to initialize..."
                sleep 20
                '''
            }
        }

        stage('Configure Broker & Server') {
            steps {
                sh """
                docker exec ${CONTAINER_NAME} bash -c '
                    set -e
                    . ${ACE_PROFILE}

                    echo "Checking integration node..."
                    if mqsilist | grep -q ${BROKER_NAME}; then
                        echo "Broker exists → restarting"
                        mqsistop ${BROKER_NAME} || true
                        mqsistart ${BROKER_NAME}
                    else
                        echo "Broker does not exist → creating"
                        mqsicreatebroker ${BROKER_NAME}
                        mqsistart ${BROKER_NAME}
                    fi

                    echo "Waiting for broker to be fully up..."
                    sleep 10

                    echo "Checking integration server..."
                    if ! mqsilist ${BROKER_NAME} | grep -q ${SERVER_NAME}; then
                        echo "Creating integration server ${SERVER_NAME}"
                        mqsicreateexecutiongroup ${BROKER_NAME} -e ${SERVER_NAME}
                    else
                        echo "Integration server already exists"
                    fi

                    echo "Final status:"
                    mqsilist ${BROKER_NAME}
                '
                """
            }
        }

        stage('Deploy BAR File') {
            steps {
                sh """
                echo "Copying BAR file into container..."
                docker cp ${BAR_FILE} ${CONTAINER_NAME}:/tmp/${BAR_FILE}

                docker exec ${CONTAINER_NAME} bash -c '
                    . ${ACE_PROFILE}

                    echo "Deploying BAR file..."
                    mqsideploy ${BROKER_NAME} \
                        -e ${SERVER_NAME} \
                        -a /tmp/${BAR_FILE} \
                        -w 120

                    echo "Deployment status:"
                    mqsilist ${BROKER_NAME} -e ${SERVER_NAME}
                '
                """
            }
        }
    }
}
