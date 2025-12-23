pipeline {
    agent any

    environment {
        IMAGE_NAME = 'moiz3388/ace13:latest'
        CONTAINER_NAME = 'aceserver'
        BROKER_NAME = 'PROD'
        SERVER_NAME = 'prodserver'
        ACE_PROFILE = '/opt/ibm/ace-13/server/bin/mqsiprofile'
        BAR_FILE = 'BVSRegFix2.bar'
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

                    sh """
                        docker exec ${CONTAINER_NAME} bash -l -c '
                            . ${ACE_PROFILE}

                            # Create broker if missing
                            if ! mqsilist | grep -q "${BROKER_NAME}"; then
                                mqsicreatebroker ${BROKER_NAME}
                            fi

                            # Start broker
                            mqsistart ${BROKER_NAME}

                            # Create server if missing
                            if ! mqsilist ${BROKER_NAME} | grep -q "${SERVER_NAME}"; then
                                mqsicreateexecutiongroup ${BROKER_NAME} -e ${SERVER_NAME}
                            fi

                            # Ensure everything is running
                            mqsistart ${BROKER_NAME}
                        '
                    """
                }
            }
        }

        stage('Deploy BAR File') {
            steps {
                sh """
                    echo "Copying BAR file into container..."
                    docker cp ${BAR_FILE} ${CONTAINER_NAME}:/tmp/${BAR_FILE}

                    docker exec ${CONTAINER_NAME} bash -l -c '
                        . ${ACE_PROFILE}

                        echo "Deploying BAR file ${BAR_FILE}..."
                        mqsideploy ${BROKER_NAME} -e ${SERVER_NAME} -a /tmp/${BAR_FILE} -w 60

                        echo "Deployment status:"
                        mqsilist ${BROKER_NAME} -e ${SERVER_NAME}
                    '
                """
            }
        }
    }
}
