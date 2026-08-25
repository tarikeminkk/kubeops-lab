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

        stage('Update GitOps Manifest') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'github-kubeops',
                        usernameVariable: 'GIT_USER',
                        passwordVariable: 'GIT_TOKEN'
                    )
                ]) {
                    sh '''
                        sed -i "s#image: ${REGISTRY}/${IMAGE_NAME}:.*#image: ${REGISTRY}/${IMAGE_NAME}:${IMAGE_TAG}#" \
                          kubernetes/deployment.yaml

                        git config user.name "KubeOps Jenkins"
                        git config user.email "jenkins@kubeops.local"

                        git add kubernetes/deployment.yaml

                        if git diff --cached --quiet; then
                            echo "No manifest changes detected."
                        else
                            git commit -m "deploy: update demo-app image to ${IMAGE_TAG}"

                            git push \
                              https://${GIT_USER}:${GIT_TOKEN}@github.com/tarikka0/kubeops-lab.git \
                              HEAD:main
                        fi
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'CI pipeline completed successfully. Deployment is managed by Argo CD.'
        }

        failure {
            echo 'CI pipeline failed.'
        }
    }
}
