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
                    docker run -d \
                      --name ${CONTAINER_NAME} \
                      -e LICENSE=accept \
                      ${IMAGE_NAME} tail -f /dev/null

                    sleep 10
                '''
            }
        }

        stage('Configure Broker & Server') {
            steps {
                sh '''
docker exec ${CONTAINER_NAME} bash -c "
set +e
. ${ACE_PROFILE}

echo 'Checking broker...'
mqsilist | grep -q ${BROKER_NAME}
if [ $? -ne 0 ]; then
    echo 'Creating broker ${BROKER_NAME}'
    mqsicreatebroker ${BROKER_NAME}
fi

echo 'Stopping broker (safe)...'
mqsistop ${BROKER_NAME} || true

echo 'Checking integration server...'
mqsilist ${BROKER_NAME} | grep -q ${SERVER_NAME}
if [ $? -ne 0 ]; then
    echo 'Creating server ${SERVER_NAME}'
    mqsicreateexecutiongroup ${BROKER_NAME} -e ${SERVER_NAME}
fi

echo 'Starting broker...'
mqsistart ${BROKER_NAME}

echo 'Broker status:'
mqsilist ${BROKER_NAME}
"
                '''
            }
        }

        stage('Deploy BAR File') {
            steps {
                sh '''
                    echo "Copying BAR file..."
                    docker cp ${BAR_FILE} ${CONTAINER_NAME}:/tmp/${BAR_FILE}

docker exec ${CONTAINER_NAME} bash -c "
set +e
. ${ACE_PROFILE}

echo 'Deploying BAR file...'
mqsideploy ${BROKER_NAME} -e ${SERVER_NAME} -a /tmp/${BAR_FILE} -w 120

echo 'Deployment status:'
mqsilist ${BROKER_NAME} -e ${SERVER_NAME}
"
                '''
            }
        }
    }
}
