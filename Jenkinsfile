environment {
    KUBE_POD = 'ace-server'
    BROKER_NAME = 'PROD'
    SERVER_NAME = 'prodserver'
    ACE_PROFILE = '/opt/ibm/ace-13/server/bin/mqsiprofile'
    BAR_FILE = 'BVSRegFix2.bar'
}

stages {
    stage('Run ACE Pod') {
        steps {
            script {
                // Apply Pod manifest
                sh "kubectl apply -f ace-pod.yaml"

                // Wait until Pod is running
                sh "kubectl wait --for=condition=Ready pod/${KUBE_POD} --timeout=120s"

                // Check pod status
                sh "kubectl get pods ${KUBE_POD} -o wide"
            }
        }
    }

    stage('Start Broker & Server') {
        steps {
            sh """
                kubectl exec ${KUBE_POD} -- bash -l -c '
                    . ${ACE_PROFILE}

                    if ! mqsilist | grep -q "${BROKER_NAME}"; then
                        mqsicreatebroker ${BROKER_NAME}
                    fi

                    if mqsilist | grep -q "${BROKER_NAME}.*running"; then
                        mqsistop ${BROKER_NAME}
                    fi

                    if ! mqsilist ${BROKER_NAME} | grep -q "${SERVER_NAME}"; then
                        mqsicreateexecutiongroup ${BROKER_NAME} -e ${SERVER_NAME}
                    fi

                    mqsistart ${BROKER_NAME}

                    echo "Final status:"
                    mqsilist ${BROKER_NAME}
                '
            """
        }
    }

    stage('Deploy BAR File') {
        steps {
            sh """
                kubectl cp ${BAR_FILE} ${KUBE_POD}:/tmp/${BAR_FILE}

                kubectl exec ${KUBE_POD} -- bash -l -c '
                    . ${ACE_PROFILE}
                    mqsideploy ${BROKER_NAME} -e ${SERVER_NAME} -a /tmp/${BAR_FILE} -w 60
                    mqsilist ${BROKER_NAME} -e ${SERVER_NAME}
                '
            """
        }
    }
}
