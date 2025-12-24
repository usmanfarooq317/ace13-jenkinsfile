pipeline {
    agent any

    environment {
        IMAGE_NAME    = 'moiz3388/ace13:latest'
        CONTAINER     = 'aceserver'
        BROKER        = 'PROD'
        SERVER        = 'prodserver'
        ACE_PROFILE   = '/opt/ibm/ace-13/server/bin/mqsiprofile'
        BAR_FILE      = 'BVSRegFix2.bar'
    }

    stages {

        stage('Clean & Start ACE Container') {
            steps {
                sh """
                echo "Stopping and removing old container if exists..."
                docker rm -f ${CONTAINER} 2>/dev/null || true

                echo "Starting fresh ACE container..."
                docker run -d --name ${CONTAINER} \\
                    -p 7600:7600 -p 7800:7800 -p 7843:7843 \\
                    -e LICENSE=accept \\
                    ${IMAGE_NAME} tail -f /dev/null
                """
            }
        }

        stage('Configure Broker & Server') {
            steps {
                sh """
                docker exec ${CONTAINER} bash -c '
                    . ${ACE_PROFILE}

                    echo "Creating broker if needed..."
                    if ! mqsilist | grep -q "${BROKER}"; then
                        mqsicreatebroker ${BROKER}
                    fi

                    echo "Starting broker..."
                    mqsistart ${BROKER}

                    echo "Creating integration server if needed..."
                    if ! mqsilist ${BROKER} | grep -q "${SERVER}"; then
                        mqsicreateexecutiongroup ${BROKER} -e ${SERVER}
                    fi

                    echo "Final status:"
                    mqsilist ${BROKER}
                '
                """
            }
        }

        stage('Deploy BAR File') {
            steps {
                sh """
                echo "Copying BAR file..."
                docker cp ${BAR_FILE} ${CONTAINER}:/tmp/${BAR_FILE}

                docker exec ${CONTAINER} bash -c '
                    . ${ACE_PROFILE}

                    echo "Deploying BAR file..."
                    mqsideploy ${BROKER} -e ${SERVER} -a /tmp/${BAR_FILE} -w 60

                    echo "Deployment result:"
                    mqsilist ${BROKER} -e ${SERVER}
                '
                """
            }
        }
    }
}
