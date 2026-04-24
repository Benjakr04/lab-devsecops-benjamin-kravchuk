pipeline {
    agent any

    options {
        timeout(time: 10, unit: 'MINUTES')
    }

    environment {
        NEXUS_URL = "http://nexus:8082"
        NEXUS_HOST = "nexus:8082"
        CREDENTIALS_ID = "nexus-credentials"
        IMAGE_NAME = "sumador"
        IMAGE_TAG = "${env.BUILD_NUMBER}"
    }

    stages {

        stage('Build Docker Image') {
            steps {
                echo "Building Docker image..."
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
            }
        }

        stage('Run Tests') {
            steps {
                sh "docker run --rm ${IMAGE_NAME}:${IMAGE_TAG} npm test"
            }
        }

        stage('Dependency Audit') {
            steps {
                echo "Running npm audit..."
                sh "docker run --rm ${IMAGE_NAME}:${IMAGE_TAG} npm audit --audit-level=critical || true"
            }
        }

        stage('Vulnerability Scan - Trivy') {
            steps {
                echo "Scanning image with Trivy..."
                sh """
                docker run --rm \
                  -v /var/run/docker.sock:/var/run/docker.sock \
                  aquasec/trivy image \
                  --severity CRITICAL \
                  --exit-code 1 \
                  ${IMAGE_NAME}:${IMAGE_TAG}
                """
            }
        }

        stage('Tag Docker Image') {
            steps {
                sh "docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${NEXUS_HOST}/${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }

        stage('Deploy Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: CREDENTIALS_ID,
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    sh """
                    echo \$NEXUS_PASS | docker login ${NEXUS_HOST} -u \$NEXUS_USER --password-stdin
                    docker push ${NEXUS_HOST}/${IMAGE_NAME}:${IMAGE_TAG}
                    docker logout ${NEXUS_HOST}
                    """
                }
            }
        }
    }

    post {
        always {
            echo "Cleaning up..."
            sh """
            docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true
            docker rmi ${NEXUS_HOST}/${IMAGE_NAME}:${IMAGE_TAG} || true
            """
        }
        success {
            echo "Pipeline completado con exito!"
        }
        failure {
            echo "Pipeline fallo. Revisar logs."
        }
    }
}