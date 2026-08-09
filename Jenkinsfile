pipeline {
    agent any

    environment {
        REGISTRY = 'harbor.kubeops.local'
        IMAGE_NAME = 'kubeops/demo-app'
        IMAGE_TAG = "${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Ansible Syntax Check') {
            steps {
                sh '''
                    cd ansible
                    ansible-playbook playbooks/ci-setup.yml --syntax-check
                    ansible-playbook playbooks/base-setup.yml --syntax-check
                    ansible-playbook playbooks/registry-setup.yml --syntax-check
                    ansible-playbook playbooks/harbor-setup.yml --syntax-check
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker build \
                      -t ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG} \
                      -t ${REGISTRY}/${IMAGE_NAME}:latest \
                      app/
                '''
            }
        }

        stage('Harbor Login') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'harbor-kubeops',
                        usernameVariable: 'HARBOR_USER',
                        passwordVariable: 'HARBOR_PASS'
                    )
                ]) {
                    sh '''
                        echo "$HARBOR_PASS" | \
                        docker login ${REGISTRY} \
                        -u "$HARBOR_USER" \
                        --password-stdin
                    '''
                }
            }
        }

        stage('Push Image') {
            steps {
                sh '''
                    docker push ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}
                    docker push ${REGISTRY}/${IMAGE_NAME}:latest
                '''
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                    export KUBECONFIG=/var/lib/jenkins/.kube/config

                    kubectl set image deployment/demo-app \
                      demo-app=${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}

                    kubectl rollout status deployment/demo-app --timeout=120s
                '''
            }
        }
        stage('Verify Deployment') {
            steps {
                sh '''
                    export KUBECONFIG=/var/lib/jenkins/.kube/config

                    kubectl get pods -l app=demo-app

                    READY_PODS=$(kubectl get pods -l app=demo-app \
                      --field-selector=status.phase=Running \
                      --no-headers | wc -l)

                    if [ "$READY_PODS" -lt 2 ]; then
                        echo "Expected 2 running demo-app pods, found $READY_PODS"
                        exit 1
                    fi

                    curl --fail --retry 5 --retry-delay 3 \
                      http://192.168.5.4:30080/
                '''
            }
        }
    }

    post {
        success {
            echo 'CI pipeline completed successfully.'
        }

        failure {
            echo 'CI pipeline failed.'
        }
    }
}
