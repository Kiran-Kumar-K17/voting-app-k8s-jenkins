pipeline {
    agent any
    
    environment {
        DOCKER_HUB_USER = 'kirankumark17'
        VOTE_IMAGE = 'kirankumark17/voting'
        RESULT_IMAGE = 'kirankumark17/result'
        WORKER_IMAGE = 'kirankumark17/worker'
        NAMESPACE = 'vote'
        K8S_SERVER = 'https://192.168.56.10:6443'
        IMAGE_TAG = "${BUILD_NUMBER}-${GIT_COMMIT}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/Kiran-Kumar-K17/voting-app-k8s-jenkins.git',
                    branch: 'main'
            }
        }
        
        stage('Login to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-hub',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                }
            }
        }
        
        stage('Detect Changes') {
            steps {
                script {
                    // Get list of changed files
                    def changedFiles = sh(script: 'git diff --name-only HEAD~1', returnStdout: true).trim()
                    echo "Changed files: ${changedFiles}"
                    
                    // Set flags based on what changed
                    env.VOTE_CHANGED = (changedFiles.contains('vote') || changedFiles.contains('voting')) ? 'true' : 'false'
                    env.RESULT_CHANGED = changedFiles.contains('result') ? 'true' : 'false'
                    env.WORKER_CHANGED = changedFiles.contains('worker') ? 'true' : 'false'
                    
                    echo "Vote changed: ${env.VOTE_CHANGED}"
                    echo "Result changed: ${env.RESULT_CHANGED}"
                    echo "Worker changed: ${env.WORKER_CHANGED}"
                }
            }
        }
        
        stage('Build and Push Images') {
            parallel {
                stage('Vote Service') {
                    when { expression { env.VOTE_CHANGED == 'true' } }
                    steps {
                        dir('example-voting-app/vote') {
                            script {
                                def imageTag = "${VOTE_IMAGE}:${IMAGE_TAG}"
                                sh """
                                echo "Building vote image: ${imageTag}"
                                docker build -t ${imageTag} .
                                docker push ${imageTag}
                                docker tag ${imageTag} ${VOTE_IMAGE}:latest
                                docker push ${VOTE_IMAGE}:latest
                                """
                                echo "✅ Vote image pushed: ${imageTag}"
                            }
                        }
                    }
                }
                
                stage('Result Service') {
                    when { expression { env.RESULT_CHANGED == 'true' } }
                    steps {
                        dir('example-voting-app/result') {
                            script {
                                def imageTag = "${RESULT_IMAGE}:${IMAGE_TAG}"
                                sh """
                                echo "Building result image: ${imageTag}"
                                docker build -t ${imageTag} .
                                docker push ${imageTag}
                                docker tag ${imageTag} ${RESULT_IMAGE}:latest
                                docker push ${RESULT_IMAGE}:latest
                                """
                                echo "✅ Result image pushed: ${imageTag}"
                            }
                        }
                    }
                }
                
                stage('Worker Service') {
                    when { expression { env.WORKER_CHANGED == 'true' } }
                    steps {
                        dir('example-voting-app/worker') {
                            script {
                                def imageTag = "${WORKER_IMAGE}:${IMAGE_TAG}"
                                sh """
                                echo "Building worker image: ${imageTag}"
                                docker build -t ${imageTag} .
                                docker push ${imageTag}
                                docker tag ${imageTag} ${WORKER_IMAGE}:latest
                                docker push ${WORKER_IMAGE}:latest
                                """
                                echo "✅ Worker image pushed: ${imageTag}"
                            }
                        }
                    }
                }
            }
        }
        
        stage('Update Kubernetes Manifests') {
            steps {
                script {
                    // Update image tags in manifests
                    sh """
                    # Update vote image if changed
                    if [ "${VOTE_CHANGED}" = "true" ]; then
                        sed -i 's|image:.*voting-app.*|image: ${VOTE_IMAGE}:${IMAGE_TAG}|g' example-voting-app/k8s-specifications/voting-deployment.yaml
                    fi
                    
                    # Update result image if changed
                    if [ "${RESULT_CHANGED}" = "true" ]; then
                        sed -i 's|image:.*result-app.*|image: ${RESULT_IMAGE}:${IMAGE_TAG}|g' example-voting-app/k8s-specifications/result-deployment.yaml
                    fi
                    
                    # Update worker image if changed
                    if [ "${WORKER_CHANGED}" = "true" ]; then
                        sed -i 's|image:.*worker-app.*|image: ${WORKER_IMAGE}:${IMAGE_TAG}|g' example-voting-app/k8s-specifications/worker-deployment.yaml
                    fi
                    """
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                withKubeConfig([credentialsId: 'k8s-token', serverUrl: K8S_SERVER]) {
                    sh """
                    # Create namespace if not exists
                    kubectl create namespace ${NAMESPACE} --dry-run=client -o yaml | kubectl apply -f -
                    
                    # Apply database and redis (infrastructure)
                    kubectl apply -f example-voting-app/k8s-specifications/db-deployment.yaml -n ${NAMESPACE}
                    kubectl apply -f example-voting-app/k8s-specifications/db-service.yaml -n ${NAMESPACE}
                    kubectl apply -f example-voting-app/k8s-specifications/redis-deployment.yaml -n ${NAMESPACE}
                    kubectl apply -f example-voting-app/k8s-specifications/redis-service.yaml -n ${NAMESPACE}
                    
                    # Apply service deployments (only updated ones will actually change)
                    kubectl apply -f example-voting-app/k8s-specifications/voting-deployment.yaml -n ${NAMESPACE}
                    kubectl apply -f example-voting-app/k8s-specifications/voting-service.yaml -n ${NAMESPACE}
                    kubectl apply -f example-voting-app/k8s-specifications/result-deployment.yaml -n ${NAMESPACE}
                    kubectl apply -f example-voting-app/k8s-specifications/result-service.yaml -n ${NAMESPACE}
                    kubectl apply -f example-voting-app/k8s-specifications/worker-deployment.yaml -n ${NAMESPACE}
                    
                    # Wait for rollouts of changed services
                    if [ "${VOTE_CHANGED}" = "true" ]; then
                        kubectl rollout status deployment/voting-app -n ${NAMESPACE} --timeout=60s
                    fi
                    
                    if [ "${RESULT_CHANGED}" = "true" ]; then
                        kubectl rollout status deployment/result-app -n ${NAMESPACE} --timeout=60s
                    fi
                    
                    if [ "${WORKER_CHANGED}" = "true" ]; then
                        kubectl rollout status deployment/worker -n ${NAMESPACE} --timeout=60s
                    fi
                    """
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                withKubeConfig([credentialsId: 'k8s-token', serverUrl: K8S_SERVER]) {
                    sh """
                    echo "=== Current Deployments ==="
                    kubectl get deployments -n ${NAMESPACE}
                    
                    echo "\\n=== Running Pods ==="
                    kubectl get pods -n ${NAMESPACE}
                    
                    echo "\\n=== Services ==="
                    kubectl get svc -n ${NAMESPACE}
                    """
                }
            }
        }
    }
    
    post {
        success {
            script {
                // Get NodePorts for access URLs
                withKubeConfig([credentialsId: 'k8s-token', serverUrl: K8S_SERVER]) {
                    def voting_port = sh(script: "kubectl get svc -n ${NAMESPACE} voting-service -o jsonpath='{.spec.ports[0].nodePort}'", returnStdout: true).trim()
                    def result_port = sh(script: "kubectl get svc -n ${NAMESPACE} result-service -o jsonpath='{.spec.ports[0].nodePort}'", returnStdout: true).trim()
                    
                    def updatedServices = []
                    if (env.VOTE_CHANGED == 'true') updatedServices.add('Vote')
                    if (env.RESULT_CHANGED == 'true') updatedServices.add('Result')
                    if (env.WORKER_CHANGED == 'true') updatedServices.add('Worker')
                    
                    echo """
                    ╔══════════════════════════════════════════════════════════╗
                    ║     ✅ DEPLOYMENT SUCCESSFUL!                            ║
                    ╚══════════════════════════════════════════════════════════╝
                    
                    📦 Images pushed to Docker Hub:
                      • ${VOTE_IMAGE}:${IMAGE_TAG}
                      • ${RESULT_IMAGE}:${IMAGE_TAG}
                      • ${WORKER_IMAGE}:${IMAGE_TAG}
                    
                    🔄 Updated services: ${updatedServices.isEmpty() ? 'None' : updatedServices.join(', ')}
                    
                    🌐 Access your application:
                      🗳️  Voting App: http://192.168.56.11:${voting_port}
                      📊 Result App: http://192.168.56.11:${result_port}
                    
                    📋 Docker Hub repos:
                      • https://hub.docker.com/r/${VOTE_IMAGE}
                      • https://hub.docker.com/r/${RESULT_IMAGE}
                      • https://hub.docker.com/r/${WORKER_IMAGE}
                    
                    ═══════════════════════════════════════════════════════════
                    """
                }
            }
        }
        
        failure {
            echo """
            ❌ DEPLOYMENT FAILED!
            
            Check the Jenkins console output for errors.
            Common issues:
            • Docker Hub login failed
            • Build errors in application code
            • Kubernetes connection issues
            • Image pull errors on nodes
            """
        }
        
        always {
            // Clean up
            sh 'docker logout || true'
        }
    }
}