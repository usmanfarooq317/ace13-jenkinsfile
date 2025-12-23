node {

    env.KUBE_POD    = 'ace-server'
    env.NAMESPACE   = 'default'
    env.BROKER_NAME = 'PROD'
    env.SERVER_NAME = 'prodserver'
    env.ACE_PROFILE = '/opt/ibm/ace-13/server/bin/mqsiprofile'
    env.BAR_FILE    = 'BVSRegFix2.bar'

    stage('Checkout') {
        checkout scm
    }

    stage('Deploy ACE Pod') {
        sh """
            kubectl apply -f ace-pod.yaml -n ${NAMESPACE}
            kubectl wait --for=condition=Ready pod/${KUBE_POD} -n ${NAMESPACE} --timeout=180s
            kubectl get pod ${KUBE_POD} -n ${NAMESPACE} -o wide
        """
    }

    stage('Create & Start Broker and Server') {
        sh """
            kubectl exec -n ${NAMESPACE} ${KUBE_POD} -- bash -l -c '
                set -e
                . ${ACE_PROFILE}

                if ! mqsilist | grep -q "${BROKER_NAME}"; then
                    echo "Creating broker ${BROKER_NAME}"
                    mqsicreatebroker ${BROKER_NAME}
                fi

                if mqsilist | grep -q "${BROKER_NAME}.*running"; then
                    echo "Stopping broker before server creation"
                    mqsistop ${BROKER_NAME}
                fi

                if ! mqsilist ${BROKER_NAME} | grep -q "${SERVER_NAME}"; then
                    echo "Creating server ${SERVER_NAME}"
                    mqsicreateexecutiongroup ${BROKER_NAME} -e ${SERVER_NAME}
                fi

                echo "Starting broker ${BROKER_NAME}"
                mqsistart ${BROKER_NAME}

                mqsilist ${BROKER_NAME}
            '
        """
    }

    stage('Deploy BAR File') {
        sh """
            kubectl cp ${BAR_FILE} ${NAMESPACE}/${KUBE_POD}:/tmp/${BAR_FILE}

            kubectl exec -n ${NAMESPACE} ${KUBE_POD} -- bash -l -c '
                set -e
                . ${ACE_PROFILE}

                echo "Deploying BAR ${BAR_FILE}"
                mqsideploy ${BROKER_NAME} -e ${SERVER_NAME} -a /tmp/${BAR_FILE} -w 60
                mqsilist ${BROKER_NAME} -e ${SERVER_NAME}
            '
        """
    }

    stage('Done') {
        echo '✅ ACE deployment completed successfully'
    }
}
