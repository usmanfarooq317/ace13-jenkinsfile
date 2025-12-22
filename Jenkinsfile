pipeline {
    agent any

    environment {
        IMAGE_NAME = 'moiz3388/ace13:latest'
        CONTAINER_NAME = 'aceserver'
        BROKER_NAME = 'PROD'
        SERVER_NAME = 'prodserver'
        ACE_PROFILE = '/home/aceuser/ace-server/server/bin/mqsiprofile' // <-- update this path
    }

    stages {
        stage('Run ACE Container') {
            steps {
                script {
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

                    // Create broker and server inside container
                    sh """
                        docker exec ${CONTAINER_NAME} bash -c '
                            # Source the correct ACE profile
                            if [ -f ${ACE_PROFILE} ]; then
                                . ${ACE_PROFILE}
                            else
                                echo "ACE profile not found at ${ACE_PROFILE}"
                                exit 1
                            fi

                            # Create broker if not exists
                            if ! mqsilist | grep -q "${BROKER_NAME}"; then
                                echo "Creating broker ${BROKER_NAME}"
                                mqsicreatebroker ${BROKER_NAME}
                            else
                                echo "Broker already exists"
                            fi

                            # Create server if not exists
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
        }
    }
}
