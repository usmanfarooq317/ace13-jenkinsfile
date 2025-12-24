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

        stage('Prepare ACE Container') {
            steps {
                script {
                    def running = sh(
                        script: "docker ps --format '{{.Names}}' | grep -w ${CONTAINER_NAME} || true",
                        returnStdout: true
                    ).trim()

                    def exists = sh(
                        script: "docker ps -a --format '{{.Names}}' | grep -w ${CONTAINER_NAME} || true",
                        returnStdout: true
                    ).trim()

                    if (running) {
                        echo "Container already running"
                    } else if (exists) {
                        echo "Starting existing container"
                        sh "docker start ${CONTAINER_NAME}"
                    } else {
                        echo "Creating new ACE container"
                        sh """
                            docker run -d --name ${CONTAINER_NAME} \
                                -p 7600:7600 \
                                -p 7800:7800 \
                                -p 7843:7843 \
                                -e LICENSE=accept \
                                ${IMAGE_NAME}
                        """
                    }
                }
            }
        }

        stage('Configure Broker & Server') {
            steps {
                sh """
docker exec ${CONTAINER_NAME} bash -lc '
    set -e
    . ${ACE_PROFILE}

    # Create broker if missing
    if ! mqsilist | grep -q "^${BROKER_NAME}"; then
        echo "Creating broker ${BROKER_NAME}"
        mqsicreatebroker ${BROKER_NAME}
    fi

    # Stop broker if running (required for server creation)
    if mqsilist | grep -q "${BROKER_NAME}.*running"; then
        echo "Stopping broker ${BROKER_NAME}"
        mqsistop ${BROKER_NAME}
    fi

    # Create integration server if missing
    if ! mqsilist ${BROKER_NAME} | grep -q "${SERVER_NAME}"; then
        echo "Creating server ${SERVER_NAME}"
        mqsicreateexecutiongroup ${BROKER_NAME} -e ${SERVER_NAME}
    fi

    # Start broker (starts server automatically)
    echo "Starting broker ${BROKER_NAME}"
    mqsistart ${BROKER_NAME}

    echo "Broker and server status:"
    mqsilist ${BROKER_NAME}
'
"""
            }
        }

        stage('Deploy BAR File') {
            steps {
                sh """
                    echo "Copying BAR file to container..."
                    docker cp ${BAR_FILE} ${CONTAINER_NAME}:/tmp/${BAR_FILE}

docker exec ${CONTAINER_NAME} bash -lc '
    set -e
    . ${ACE_PROFILE}

    echo "Deploying BAR file ${BAR_FILE}"
    mqsideploy ${BROKER_NAME} \
        -e ${SERVER_NAME} \
        -a /tmp/${BAR_FILE} \
        -w 120

    echo "Deployment result:"
    mqsilist ${BROKER_NAME} -e ${SERVER_NAME}
'
"""
            }
        }
    }
}
