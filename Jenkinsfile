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
        withCredentials([string(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_CONTENT')]) {
            sh '''
                set -e
                echo "$KUBECONFIG_CONTENT" > kubeconfig
                export KUBECONFIG=$PWD/kubeconfig

                kubectl apply -f ace-pod.yaml -n default
                kubectl wait --for=condition=Ready pod/ace-server -n default --timeout=180s
                kubectl get pod ace-server -n default -o wide
            '''
        }
    }

    stage('Create & Start Broker and Server') {
        withCredentials([string(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_CONTENT')]) {
            sh '''
                set -e
                echo "$KUBECONFIG_CONTENT" > kubeconfig
                export KUBECONFIG=$PWD/kubeconfig

                kubectl exec -n default ace-server -- bash -l -c "
                    set -e
                    . /opt/ibm/ace-13/server/bin/mqsiprofile

                    if ! mqsilist | grep -q PROD; then
                        mqsicreatebroker PROD
                    fi

                    if mqsilist | grep -q 'PROD.*running'; then
                        mqsistop PROD
                    fi

                    if ! mqsilist PROD | grep -q prodserver; then
                        mqsicreateexecutiongroup PROD -e prodserver
                    fi

                    mqsistart PROD
                    mqsilist PROD
                "
            '''
        }
    }

    stage('Deploy BAR File') {
        withCredentials([string(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_CONTENT')]) {
            sh '''
                set -e
                echo "$KUBECONFIG_CONTENT" > kubeconfig
                export KUBECONFIG=$PWD/kubeconfig

                kubectl cp BVSRegFix2.bar default/ace-server:/tmp/BVSRegFix2.bar

                kubectl exec -n default ace-server -- bash -l -c "
                    . /opt/ibm/ace-13/server/bin/mqsiprofile
                    mqsideploy PROD -e prodserver -a /tmp/BVSRegFix2.bar -w 60
                    mqsilist PROD -e prodserver
                "
            '''
        }
    }

    stage('Done') {
        echo '✅ ACE deployment completed successfully'
    }
}
