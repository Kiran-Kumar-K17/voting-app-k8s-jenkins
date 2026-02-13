pipeline {
    agent any

    environment {
        DOCKER_USER = "kirankumark17"
        DOCKER_TAG  = "${BUILD_NUMBER}"
    }

    // ── Global timeout: kill the entire pipeline after 30 minutes ──
    options {
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
    }

    stages {

        // --------------------------------------------------
        // 1️⃣ Checkout
        // --------------------------------------------------
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git fetch --all'
            }
        }

        // --------------------------------------------------
        // 2️⃣ Detect Changes
        // --------------------------------------------------
        stage('Detect Changes') {
            steps {
                script {

                    def currentCommit = sh(
                        script: "git rev-parse HEAD",
                        returnStdout: true
                    ).trim()

                    echo "Current Commit: ${currentCommit}"

                    def lastCommit = ""

                    if (fileExists('.last_successful_commit')) {
                        lastCommit = readFile('.last_successful_commit').trim()
                        echo "Last Successful Commit: ${lastCommit}"
                    } else {
                        echo "No previous successful build found. Building everything."
                        env.BUILD_VOTING = 'true'
                        env.BUILD_WORKER = 'true'
                        env.BUILD_RESULT = 'true'
                        return
                    }

                    if (lastCommit == currentCommit) {
                        echo "No new commit detected. Skipping build."
                        env.BUILD_VOTING = 'false'
                        env.BUILD_WORKER = 'false'
                        env.BUILD_RESULT = 'false'
                        return
                    }

                    def changedFiles = sh(
                        script: "git diff --name-only ${lastCommit} ${currentCommit}",
                        returnStdout: true
                    ).trim()

                    echo "Changed files:\n${changedFiles}"

                    env.BUILD_VOTING = changedFiles.contains('vote/')   ? 'true' : 'false'
                    env.BUILD_WORKER = changedFiles.contains('worker/') ? 'true' : 'false'
                    env.BUILD_RESULT = changedFiles.contains('result/') ? 'true' : 'false'
                }
            }
        }

        // --------------------------------------------------
        // 3️⃣ No Changes
        // --------------------------------------------------
        stage('No Changes') {
            when {
                expression {
                    env.BUILD_VOTING == 'false' &&
                    env.BUILD_WORKER == 'false' &&
                    env.BUILD_RESULT == 'false'
                }
            }
            steps {
                echo "No service changes detected. Pipeline exiting."
            }
        }

        // --------------------------------------------------
        // 4️⃣ Docker Login
        // --------------------------------------------------
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

        // --------------------------------------------------
        // 5️⃣ Build & Deploy (Parallel)
        // --------------------------------------------------
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
                    // ── Per-stage timeout: kills hung kubectl after 10 min ──
                    options { timeout(time: 10, unit: 'MINUTES') }
                    when { expression { env.BUILD_VOTING == 'true' } }
                    steps {
                        deployService("vote", "voting")
                    }
                }

                stage('Worker') {
                    options { timeout(time: 10, unit: 'MINUTES') }
                    when { expression { env.BUILD_WORKER == 'true' } }
                    steps {
                        deployService("worker", "worker")
                    }
                }

                stage('Result') {
                    options { timeout(time: 10, unit: 'MINUTES') }
                    when { expression { env.BUILD_RESULT == 'true' } }
                    steps {
                        deployService("result", "result")
                    }
                }
            }
        }
    }

    // --------------------------------------------------
    // POST ACTIONS
    // --------------------------------------------------
    post {

        always {
            sh 'docker logout || true'
        }

        success {
            script {
                def currentCommit = sh(
                    script: "git rev-parse HEAD",
                    returnStdout: true
                ).trim()

                writeFile file: '.last_successful_commit', text: currentCommit

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
            echo "Pipeline failed."
        }

        aborted {
            echo "Pipeline was aborted. Check for orphaned kubectl processes on the agent."
        }
    }
}


// ==========================================================
// 🔥 Deploy Function  —  KUBECONFIG passed via env, not interpolation
// ==========================================================
def deployService(folderName, deploymentName) {

    withCredentials([file(
        credentialsId: 'kubeconfig',
        variable: 'KUBECONFIG_FILE'   // renamed to avoid Groovy interpolation warning
    )]) {

        // Use single-quotes so the shell expands $KUBECONFIG_FILE, not Groovy
        sh '''
            export KUBECONFIG="$KUBECONFIG_FILE"

            docker build \
                -t ''' + env.DOCKER_USER + '/' + deploymentName + ':' + env.DOCKER_TAG + ''' \
                -t ''' + env.DOCKER_USER + '/' + deploymentName + ''':latest \
                ./example-voting-app/''' + folderName + '''

            docker push ''' + env.DOCKER_USER + '/' + deploymentName + ':' + env.DOCKER_TAG + '''
            docker push ''' + env.DOCKER_USER + '/' + deploymentName + ''':latest

            # Update the image — record current rollout history before we start
            kubectl set image deployment/''' + deploymentName + ' ' + deploymentName + '=' + env.DOCKER_USER + '/' + deploymentName + ':' + env.DOCKER_TAG + '''

            # Wait up to 2 minutes for rollout; auto-rollback on failure
            if ! kubectl rollout status deployment/''' + deploymentName + ''' --timeout=120s; then
                echo "Rollout failed for ''' + deploymentName + '''. Rolling back..."
                kubectl rollout undo deployment/''' + deploymentName + '''
                kubectl rollout status deployment/''' + deploymentName + ''' --timeout=60s
                exit 1
            fi

            echo "''' + deploymentName + ''' deployed successfully."
        '''
    }
}