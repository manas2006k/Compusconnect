
pipeline {
    agent any

    environment {
        DOCKER_IMAGE = 'ganeshg06/skillbridge'
        K8S_SERVER = 'https://192.168.49.2:8443'
        SUPABASE_URL = 'https://eldmwjkolqqdhfqcawvg.supabase.co'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        
        stage('Build Docker Image') {
            steps {
                script {
                    env.IMAGE_TAG = "${env.BUILD_NUMBER}"
                }

                withCredentials([
                    string(
                        credentialsId: 'supabase-anon-key',
                        variable: 'SUPABASE_ANON_KEY'
                    )
                ]) {
                    sh '''
                        set +x
                        docker build \
                          --build-arg NEXT_PUBLIC_SUPABASE_URL="$SUPABASE_URL" \
                          --build-arg NEXT_PUBLIC_SUPABASE_ANON_KEY="$SUPABASE_ANON_KEY" \
                          -t "$DOCKER_IMAGE:$IMAGE_TAG" .
                    '''
                }
            }
        }


        stage('Push to Docker Hub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_TOKEN'
                    )
                ]) {
                    sh '''
                        set +x
                        echo "$DOCKER_TOKEN" | docker login \
                          -u "$DOCKER_USER" --password-stdin
                        docker push "$DOCKER_IMAGE:$IMAGE_TAG"
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    string(
                        credentialsId: 'k8s-deployer-token',
                        variable: 'K8S_TOKEN'
                    )
                ]) {
                    sh '''
                        set +x
                        export KUBECONFIG="$WORKSPACE/.kubeconfig"

                        kubectl config set-cluster minikube \
                          --server="$K8S_SERVER" \
                          --certificate-authority=/tmp/minikube-ca.crt \
                          --embed-certs=true

                        kubectl config set-credentials jenkins-deployer \
                          --token="$K8S_TOKEN"

                        kubectl config set-context skillbridge \
                          --cluster=minikube \
                          --user=jenkins-deployer \
                          --namespace=default

                        kubectl config use-context skillbridge

                        kubectl set image deployment/skillbridge \
                          skillbridge="$DOCKER_IMAGE:$IMAGE_TAG"

                        kubectl rollout status deployment/skillbridge \
                          --timeout=180s

                        kubectl get pods -l app=skillbridge
                    '''
                }
            }
        }
    }

    post {
        always {
            sh 'rm -f "$WORKSPACE/.kubeconfig"'
        }
        success {
            echo 'SkillBridge image pushed and deployed successfully!'
        }
        failure {
            echo 'Pipeline failed. Check the Console Output for the failing stage.'
        }
    }
}
