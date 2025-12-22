pipeline {
    agent any

    environment {
        IMAGE_NAME = 'moiz3388/ace13:latest'
        CONTAINER_NAME = 'aceserver'
        BROKER_NAME = 'PROD'
        SERVER_NAME = 'prodserver'
        ACE_PROFILE = '/opt/ibm/ace-13/server/bin/mqsiprofile'
    }

    stages {
        stage('Run ACE Container') {
            steps {
                script {
                    // Check if container exists
                    def containerExists = sh(
                        script: "docker ps -a --format '{{.Names}}' | grep -w ${CONTAINER_NAME} || true",
                        returnStdout: true
                    ).trim()

                    if (containerExists) {
                        echo "Container exists. Restarting..."
                        sh "docker restart ${CONTAINER_NAME}"
                    } else {
                        echo "Container does not exist. Creating..."
                        sh """
                            docker run -d --name ${CONTAINER_NAME} \
                                -p 7600:7600 -p 7800:7800 -p 7843:7843 \
                                --env LICENSE=accept \
                                ${IMAGE_NAME} tail -f /dev/null
                        """
                    }

                    // Ensure broker and server are created and running
                    sh """
                        docker exec ${CONTAINER_NAME} bash -c '
                            # Source ACE environment
                            . ${ACE_PROFILE}

                            # Create broker if not exists
                            if ! mqsilist | grep -q "${BROKER_NAME}"; then
                                echo "Creating broker ${BROKER_NAME}"
                                mqsicreatebroker ${BROKER_NAME}
                            else
                                echo "Broker already exists"
                            fi

                            # Create execution group/server if not exists
                            if ! mqsilist ${BROKER_NAME} | grep -q "${SERVER_NAME}"; then
                                echo "Creating server ${SERVER_NAME}"
                                mqsicreateexecutiongroup ${BROKER_NAME} -e ${SERVER_NAME}
                            else
                                echo "Server already exists"
                            fi

                            # Start broker (this also starts all servers)
                            mqsistart ${BROKER_NAME}

                            # Confirm status
                            echo "Integration node and server status:"
                            mqsilist ${BROKER_NAME}
                        '
                    """
                }
            }
        }
    }
}
