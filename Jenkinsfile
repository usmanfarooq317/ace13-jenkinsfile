pipeline {
    agent any

    environment {
        KUBE_POD    = 'ace-server'
        NAMESPACE   = 'default'
        BROKER_NAME = 'PROD'
        SERVER_NAME = 'prodserver'
        ACE_PROFILE = '/opt/ibm/ace-13/server/bin/mqsiprofile'
        BAR_FILE    = 'BVSRegFix2.bar'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Test K8s Access') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh '''
                        set -e
                        export KUBECONFIG=$KUBECONFIG
                        echo "✅ Kubernetes nodes:"
                        kubectl get nodes
                        echo "✅ Current pods in namespace ${NAMESPACE}:"
                        kubectl get pods -n ${NAMESPACE}
                    '''
                }
            }
        }

        stage('Deploy ACE Pod') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh '''
                        set -e
                        export KUBECONFIG=$KUBECONFIG

                        echo "🔹 Deploying ACE pod..."
                        kubectl apply -f ace-pod.yaml -n ${NAMESPACE}

                        echo "⏳ Waiting for pod to be ready..."
                        kubectl wait --for=condition=Ready pod/${KUBE_POD} -n ${NAMESPACE} --timeout=180s

                        echo "✅ Pod status:"
                        kubectl get pod ${KUBE_POD} -n ${NAMESPACE} -o wide
                    '''
                }
            }
        }

        stage('Create & Start Broker and Server') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh '''
                        set -e
                        export KUBECONFIG=$KUBECONFIG

                        echo "🔹 Setting up broker and server..."
                        kubectl exec -n ${NAMESPACE} ${KUBE_POD} -- bash -l -c "
                            source ${ACE_PROFILE}

                            # Create broker if not exists
                            if ! mqsilist | grep -q ${BROKER_NAME}; then
                                echo 'Creating broker ${BROKER_NAME}...'
                                mqsicreatebroker ${BROKER_NAME}
                            fi

                            # Stop broker if running
                            if mqsilist | grep -q '${BROKER_NAME}.*running'; then
                                echo 'Stopping running broker ${BROKER_NAME}...'
                                mqsistop ${BROKER_NAME}
                            fi

                            # Create execution group if not exists
                            if ! mqsilist ${BROKER_NAME} | grep -q ${SERVER_NAME}; then
                                echo 'Creating execution group ${SERVER_NAME}...'
                                mqsicreateexecutiongroup ${BROKER_NAME} -e ${SERVER_NAME}
                            fi

                            # Start broker
                            echo 'Starting broker ${BROKER_NAME}...'
                            mqsistart ${BROKER_NAME}

                            echo '✅ Broker and server setup complete.'
                            mqsilist ${BROKER_NAME}
                        "
                    '''
                }
            }
        }

        stage('Deploy BAR File') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh '''
                        set -e
                        export KUBECONFIG=$KUBECONFIG

                        echo "🔹 Copying BAR file to pod..."
                        kubectl cp ${BAR_FILE} ${NAMESPACE}/${KUBE_POD}:/tmp/${BAR_FILE}

                        echo "🔹 Deploying BAR file..."
                        kubectl exec -n ${NAMESPACE} ${KUBE_POD} -- bash -l -c "
                            source ${ACE_PROFILE}
                            mqsideploy ${BROKER_NAME} -e ${SERVER_NAME} -a /tmp/${BAR_FILE} -w 60
                            echo '✅ BAR deployment complete.'
                            mqsilist ${BROKER_NAME} -e ${SERVER_NAME}
                        "
                    '''
                }
            }
        }

        stage('Done') {
            steps {
                echo '🎉 ACE deployment completed successfully.'
            }
        }
    }
}
