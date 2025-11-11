pipeline {
    agent any

    environment {
        DOCKER_IMAGE_NAME = 'springboot-demo'
        DOCKER_REGISTRY = 'dnwls16071/'
        IMAGE_TAG = "${env.BUILD_NUMBER}"
        FULL_IMAGE_NAME = "${DOCKER_REGISTRY}${DOCKER_IMAGE_NAME}:${IMAGE_TAG}"

        KUBECONFIG = credentials('kubeconfig')
        NAMESPACE = 'springboot-demo'
    }

    stages {
        stage('Checkout') {
            steps {
                echo '=== Checking out source code ==='
                checkout scm
                script {
                    env.GIT_COMMIT_MSG = sh(
                        script: 'git log -1 --pretty=%B',
                        returnStdout: true
                    ).trim()
                    env.GIT_AUTHOR = sh(
                        script: 'git log -1 --pretty=%an',
                        returnStdout: true
                    ).trim()
                }
            }
        }

        stage('Build') {
            steps {
                echo '=== Building with Gradle ==='
                sh '''
                    chmod +x gradlew
                    ./gradlew clean build -x test --no-daemon
                '''
            }
        }

        stage('Test') {
            steps {
                echo '=== Running Tests ==='
                sh './gradlew test --no-daemon'
            }
            post {
                always {
                    junit '**/build/test-results/test/*.xml'
                    archiveArtifacts artifacts: '**/build/libs/*.jar', fingerprint: true
                }
            }
        }

        stage('Docker Build') {
            steps {
                echo '=== Building Docker Image ==='
                script {
                    sh """
                        docker build -t ${FULL_IMAGE_NAME} .
                        docker tag ${FULL_IMAGE_NAME} ${DOCKER_REGISTRY}${DOCKER_IMAGE_NAME}:latest
                    """
                }
            }
        }

        stage('Docker Push') {
            when {
                expression { env.DOCKER_REGISTRY != '' }
            }
            steps {
                echo '=== Pushing Docker Image ==='
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'docker-hub-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh """
                            echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin
                            docker push ${FULL_IMAGE_NAME}
                            docker push ${DOCKER_REGISTRY}${DOCKER_IMAGE_NAME}:latest
                        """
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                echo '=== Deploying to Kubernetes ==='
                script {
                    sh """
                        kubectl apply -f deploy/namespace.yml
                        kubectl apply -f deploy/configMap.yml

                        export IMAGE_TAG=${IMAGE_TAG}
                        export DOCKER_REGISTRY=${DOCKER_REGISTRY}
                        export DOCKER_IMAGE_NAME=${DOCKER_IMAGE_NAME}
                        envsubst < deploy/deployment.yml | kubectl apply -f -

                        kubectl apply -f deploy/service.yml
                        kubectl apply -f deploy/hpa.yml
                        kubectl rollout status deployment/springboot-app -n ${NAMESPACE} --timeout=5m
                        kubectl get pods -n ${NAMESPACE} -l app=springboot-app
                        kubectl get svc -n ${NAMESPACE}
                    """
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                echo '=== Verifying Deployment ==='
                script {
                    sh """
                        # Pod가 Ready 상태인지 확인
                        kubectl wait --for=condition=ready pod -l app=springboot-app -n ${NAMESPACE} --timeout=5m

                        # Service Endpoint 확인
                        kubectl get endpoints -n ${NAMESPACE}

                        # 최근 이벤트 확인
                        kubectl get events -n ${NAMESPACE} --sort-by='.lastTimestamp' | tail -10
                    """
                }
            }
        }
    }

    post {
        success {
            echo '=== Pipeline Succeeded ==='
            echo "Deployed ${FULL_IMAGE_NAME} to Kubernetes"
            echo "Commit: ${env.GIT_COMMIT_MSG} by ${env.GIT_AUTHOR}"
        }
        failure {
            echo '=== Pipeline Failed ==='
            script {
                sh """
                    echo "Checking pod status..."
                    kubectl get pods -n ${NAMESPACE} -l app=springboot-app || true

                    echo "Checking recent events..."
                    kubectl get events -n ${NAMESPACE} --sort-by='.lastTimestamp' | tail -20 || true
                """
            }
        }
        always {
            echo '=== Cleaning up ==='
            cleanWs()
        }
    }
}