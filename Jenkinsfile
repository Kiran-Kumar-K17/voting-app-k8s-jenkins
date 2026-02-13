pipeline {
    agent any

    environment {
        DOCKER_USER = "kirankumark17"
        DOCKER_TAG  = "${BUILD_NUMBER}"
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
        // 2️⃣ Detect Changes (Safe Version)
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

    // --------------------------------------------------
    // POST ACTIONS
    // --------------------------------------------------
    post {

        always {
            sh 'docker logout || true'
        }

        success {
            script {

                // Save current commit for next comparison
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
            echo "Pipeline failed. Rollback executed if needed."
        }
    }
}


// ==========================================================
// 🔥 Deploy Function
// ==========================================================
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
