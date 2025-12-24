pipeline {
    agent any

    environment {
        IMAGE_NAME     = 'moiz3388/ace13:latest'
        CONTAINER_NAME = 'aceserver'
        SERVER_NAME    = 'prodserver'
        ACE_PROFILE    = '/opt/ibm/ace-13/server/bin/mqsiprofile'
        BAR_FILE       = 'BVSRegFix2.bar'
    }

    stages {

        stage('Clean & Start ACE Container') {
            steps {
                sh '''
                    echo "Removing old container if exists..."
                    docker rm -f ${CONTAINER_NAME} || true

                    echo "Starting ACE container..."
                    docker run -d --name ${CONTAINER_NAME} \
                        -e LICENSE=accept \
                        ${IMAGE_NAME}

                    echo "Waiting for ACE to initialize..."
                    sleep 15
                '''
            }
        }

        stage('Create Integration Server') {
            steps {
                sh '''
                    docker exec ${CONTAINER_NAME} bash -c '
                        set -e
                        . ${ACE_PROFILE}

                        echo "Listing integration node:"
                        mqsilist

                        echo "Checking integration server..."
                        if ! mqsilist | grep -q ${SERVER_NAME}; then
                            echo "Creating integration server ${SERVER_NAME}"
                            mqsicreateexecutiongroup --integration-node PROD -e ${SERVER_NAME}
                        else
                            echo "Integration server already exists"
                        fi

                        echo "Final status:"
                        mqsilist
                    '
                '''
            }
        }

        stage('Deploy BAR File') {
            steps {
                sh '''
                    echo "Copying BAR file into container..."
                    docker cp ${BAR_FILE} ${CONTAINER_NAME}:/tmp/${BAR_FILE}

                    docker exec ${CONTAINER_NAME} bash -c '
                        set -e
                        . ${ACE_PROFILE}

                        echo "Deploying BAR file..."
                        mqsideploy PROD \
                            -e ${SERVER_NAME} \
                            -a /tmp/${BAR_FILE} \
                            -w 120

                        echo "Deployment complete"
                        mqsilist PROD -e ${SERVER_NAME}
                    '
                '''
            }
        }
    }
}
