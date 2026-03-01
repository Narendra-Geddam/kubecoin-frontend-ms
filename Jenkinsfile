pipeline {
    agent { label 'docker-host' }

    environment {
        DOCKER_CREDENTIALS = 'docker-creds'
        GIT_CREDENTIALS = 'git-creds'
        DOCKER_REGISTRY = 'your-private-registry.example.com'

        IMAGE_NAME = 'kubecoin-frontend'
        IMAGE_REPOSITORY = "${DOCKER_REGISTRY}/${IMAGE_NAME}"
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        IMAGE_FULL = "${IMAGE_REPOSITORY}:${IMAGE_TAG}"
        IMAGE_LATEST = "${IMAGE_REPOSITORY}:latest"

        HELM_REPO_URL = 'https://github.com/Narendra-Geddam/kubecoin-helm.git'
        HELM_BRANCH = 'main'
        HELM_VALUES_FILE = 'kubecoin/values.yaml'
    }

    triggers {
        githubPush()
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Image') {
            steps {
                sh 'docker build -t ${IMAGE_FULL} -t ${IMAGE_LATEST} .'
            }
        }

        stage('Login to Registry') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: DOCKER_CREDENTIALS,
                    usernameVariable: 'DOCKER_USERNAME',
                    passwordVariable: 'DOCKER_PASSWORD'
                )]) {
                    sh 'echo "$DOCKER_PASSWORD" | docker login ${DOCKER_REGISTRY} -u "$DOCKER_USERNAME" --password-stdin'
                }
            }
        }

        stage('Push Images') {
            steps {
                sh '''
                    docker push ${IMAGE_FULL}
                    docker push ${IMAGE_LATEST}
                '''
            }
        }

        stage('Update Helm Image Tag') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: GIT_CREDENTIALS,
                    usernameVariable: 'GIT_USERNAME',
                    passwordVariable: 'GIT_PASSWORD'
                )]) {
                    sh '''
                        set -e
                        rm -rf helm-repo

                        CLEAN_HELM_URL=${HELM_REPO_URL#https://}
                        git clone -b ${HELM_BRANCH} https://${GIT_USERNAME}:${GIT_PASSWORD}@${CLEAN_HELM_URL} helm-repo

                        sed -i '/^frontend:/,/^[^ ]/ s|^\(    repository: \).*|\1'"${IMAGE_REPOSITORY}"'|' helm-repo/${HELM_VALUES_FILE}
                        sed -i '/^frontend:/,/^[^ ]/ s|^\(    tag: \).*|\1"'"${IMAGE_TAG}"'"|' helm-repo/${HELM_VALUES_FILE}

                        cd helm-repo
                        git config user.name "jenkins"
                        git config user.email "jenkins@local"
                        git add ${HELM_VALUES_FILE}

                        if git diff --cached --quiet; then
                          echo "No Helm changes to commit."
                          exit 0
                        fi

                        git commit -m "ci(frontend): update image to ${IMAGE_TAG}"
                        git push origin ${HELM_BRANCH}
                    '''
                }
            }
        }
    }

    post {
        always {
            sh 'docker logout ${DOCKER_REGISTRY} || true'
            sh 'rm -rf helm-repo || true'
        }
    }
}