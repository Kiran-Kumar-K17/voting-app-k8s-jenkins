pipeline {
    agent any

    environment {
    DOCKER_USER = "kirankumark17"
    DOCKER_TAG  = "${BUILD_NUMBER}"
}


    stages {

        // ─────────────────────────────────────────────
        // 1. Checkout
        // ─────────────────────────────────────────────
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git fetch --all'
            }
        }

        // ─────────────────────────────────────────────
        // 2. Detect Changes
        //    Guards against the first-ever build where
        //    HEAD~1 does not exist yet.
        // ─────────────────────────────────────────────
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

                    env.BUILD_VOTING = changedFiles.contains('voting/')  ? 'true' : 'false'
                    env.BUILD_WORKER = changedFiles.contains('worker/')  ? 'true' : 'false'
                    env.BUILD_RESULT = changedFiles.contains('result/')  ? 'true' : 'false'
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

        // ─────────────────────────────────────────────
        // 3. Docker Login
        // ─────────────────────────────────────────────
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
                    sh 'echo $DOCKER_PASSWORD | docker login -u $DOCKER_USERNAME --password-stdin'
                }
            }
        }

        // ─────────────────────────────────────────────
        // 4. Build, Push & Deploy — run in parallel
        //    Each service:
        //      • Skipped when no relevant files changed
        //      • Pushes both :<build-number> and :latest
        //      • Verifies the rollout; undoes on failure
        // ─────────────────────────────────────────────
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
                        withCredentials([file(
                            credentialsId: 'kubeconfig',
                            variable: 'KUBECONFIG'
                        )]) {
                            sh """
                                docker build \
                                    -t ${DOCKER_USER}/voting:${DOCKER_TAG} \
                                    -t ${DOCKER_USER}/voting:latest \
                                    ./voting

                                docker push ${DOCKER_USER}/voting:${DOCKER_TAG}
                                docker push ${DOCKER_USER}/voting:latest

                                kubectl set image deployment/voting \
                                    voting=${DOCKER_USER}/voting:${DOCKER_TAG}

                                kubectl rollout status deployment/voting --timeout=120s \
                                    || (kubectl rollout undo deployment/voting && exit 1)
                            """
                        }
                    }
                }

                stage('Worker') {
                    when { expression { env.BUILD_WORKER == 'true' } }
                    steps {
                        withCredentials([file(
                            credentialsId: 'kubeconfig',
                            variable: 'KUBECONFIG'
                        )]) {
                            sh """
                                docker build \
                                    -t ${DOCKER_USER}/worker:${DOCKER_TAG} \
                                    -t ${DOCKER_USER}/worker:latest \
                                    ./worker

                                docker push ${DOCKER_USER}/worker:${DOCKER_TAG}
                                docker push ${DOCKER_USER}/worker:latest

                                kubectl set image deployment/worker \
                                    worker=${DOCKER_USER}/worker:${DOCKER_TAG}

                                kubectl rollout status deployment/worker --timeout=120s \
                                    || (kubectl rollout undo deployment/worker && exit 1)
                            """
                        }
                    }
                }

                stage('Result') {
                    when { expression { env.BUILD_RESULT == 'true' } }
                    steps {
                        withCredentials([file(
                            credentialsId: 'kubeconfig',
                            variable: 'KUBECONFIG'
                        )]) {
                            sh """
                                docker build \
                                    -t ${DOCKER_USER}/result:${DOCKER_TAG} \
                                    -t ${DOCKER_USER}/result:latest \
                                    ./result

                                docker push ${DOCKER_USER}/result:${DOCKER_TAG}
                                docker push ${DOCKER_USER}/result:latest

                                kubectl set image deployment/result \
                                    result=${DOCKER_USER}/result:${DOCKER_TAG}

                                kubectl rollout status deployment/result --timeout=120s \
                                    || (kubectl rollout undo deployment/result && exit 1)
                            """
                        }
                    }
                }

            }
        }
    }

    // ─────────────────────────────────────────────
    // Post-build actions
    //   always  → log out of Docker regardless of outcome
    //   success → mark the image tags in the build description
    //   failure → surface a clear message (hook in Slack/email here)
    // ─────────────────────────────────────────────
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
                if (deployed) {
                    currentBuild.description = "Deployed: ${deployed.join(', ')}"
                } else {
                    currentBuild.description = "No services changed — nothing deployed"
                }
            }
        }
        failure {
            echo "Pipeline failed. Check logs above. Any failed rollout has been automatically undone."
            // Add Slack / email notification here, e.g.:
            // slackSend channel: '#deployments', color: 'danger',
            //           message: "Build ${BUILD_NUMBER} failed: ${BUILD_URL}"
        }
    }
}