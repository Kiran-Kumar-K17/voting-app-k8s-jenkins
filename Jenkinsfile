pipeline {
    agent any

    environment {
        DOCKER_USER = "kirankumark17"
        DOCKER_TAG  = "${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                sh 'git fetch --all'
            }
        }

        stage('Detect Changes') {
            steps {
                script {
                    def changedFiles = sh(
                        script: """
                            if git rev-parse HEAD~1 > /dev/null 2>&1; then
                                git diff --name-only HEAD~1 HEAD
                            else
                                git diff --name-only HEAD
                            fi
                        """,
                        returnStdout: true
                    ).trim()

                    echo "Changed files:\n${changedFiles}"

                    env.BUILD_VOTING = changedFiles.contains('vote/')   ? 'true' : 'false'
                    env.BUILD_WORKER = changedFiles.contains('worker/') ? 'true' : 'false'
                    env.BUILD_RESULT = changedFiles.contains('result/') ? 'true' : 'false'
                }
            }
        }

        stage('No Changes') {
            when {
                expression {
                    env.BUILD_VOTING == 'false' &&
                    env.BUILD_WORKER == 'false' &&
                    env.BUILD_RESULT == 'false'
                }
            }
            steps {
                echo "No service code changes detected."
            }
        }

        stage('Docker Login') {
            when {
                expression {
                    env.BUILD_VOTING == 'true' ||
                    env.BUILD_WORKER == 'true' ||
                    env.BUILD_RESULT == 'true'
                }
            }
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USERNAME',
                    passwordVariable: 'DOCKER_PASSWORD'
                )]) {
                    sh '''
                        echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin
                    '''
                }
            }
        }

        stage('Build & Deploy Services') {
            when {
                expression {
                    env.BUILD_VOTING == 'true' ||
                    env.BUILD_WORKER == 'true' ||
                    env.BUILD_RESULT == 'true'
                }
            }

            parallel {

                stage('Voting') {
                    when { expression { env.BUILD_VOTING == 'true' } }
                    steps {
                        deployService("vote", "voting")
                    }
                }

                stage('Worker') {
                    when { expression { env.BUILD_WORKER == 'true' } }
                    steps {
                        deployService("worker", "worker")
                    }
                }

                stage('Result') {
                    when { expression { env.BUILD_RESULT == 'true' } }
                    steps {
                        deployService("result", "result")
                    }
                }
            }
        }
    }

    post {
        always {
            sh 'docker logout || true'
        }

        success {
            script {
                def deployed = []
                if (env.BUILD_VOTING == 'true') deployed << "voting:${DOCKER_TAG}"
                if (env.BUILD_WORKER == 'true') deployed << "worker:${DOCKER_TAG}"
                if (env.BUILD_RESULT == 'true') deployed << "result:${DOCKER_TAG}"

                currentBuild.description = deployed ?
                    "Deployed: ${deployed.join(', ')}" :
                    "No services changed — nothing deployed"
            }
        }

        failure {
            echo "Pipeline failed. Rollback executed if needed."
        }
    }
}


def deployService(folderName, deploymentName) {

    withCredentials([file(
        credentialsId: 'kubeconfig',
        variable: 'KUBECONFIG'
    )]) {

        sh """
            export KUBECONFIG=${KUBECONFIG}

            docker build \
                -t ${DOCKER_USER}/${deploymentName}:${DOCKER_TAG} \
                -t ${DOCKER_USER}/${deploymentName}:latest \
                ./example-voting-app/${folderName}

            docker push ${DOCKER_USER}/${deploymentName}:${DOCKER_TAG}
            docker push ${DOCKER_USER}/${deploymentName}:latest

            kubectl set image deployment/${deploymentName} \
                ${deploymentName}=${DOCKER_USER}/${deploymentName}:${DOCKER_TAG}

            kubectl rollout status deployment/${deploymentName} --timeout=120s \
                || (kubectl rollout undo deployment/${deploymentName} && exit 1)
        """
    }
}
