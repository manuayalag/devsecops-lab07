pipeline {
    agent any

    options {
        timeout(time: 10, unit: 'MINUTES')
    }

    environment {
        IMAGE_NAME = "sumador"
        IMAGE_TAG  = "${env.BUILD_NUMBER}"
        CREDENTIALS_ID = "nexus-credentials"
        NEXUS_HOST = "nexus:8083"
        NEXUS_URL  = "http://nexus:8083"
        NEXUS_REPO = "docker-hosted"
        FULL_IMAGE = "${NEXUS_HOST}/${IMAGE_NAME}:${IMAGE_TAG}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                echo "Código descargado. Build #${IMAGE_TAG}"
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "Construyendo imagen Docker ${IMAGE_NAME}:${IMAGE_TAG}..."
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
            }
        }

        stage('Dependency Scan - npm audit') {
            steps {
                echo "Analizando dependencias con npm audit..."
                sh "docker run --rm ${IMAGE_NAME}:${IMAGE_TAG} sh -c 'npm audit --audit-level=critical'"
            }
        }

        stage('Run Tests') {
            steps {
                echo "Ejecutando tests unitarios..."
                sh "docker run --rm ${IMAGE_NAME}:${IMAGE_TAG} npm test"
            }
        }

        stage('Vulnerability Scan - Trivy') {
            steps {
                echo "Escaneando imagen Docker con Trivy..."
                sh """
                    docker run --rm \
                        -v /var/run/docker.sock:/var/run/docker.sock \
                        aquasec/trivy image \
                        --severity CRITICAL \
                        --exit-code 1 \
                        --no-progress \
                        ${IMAGE_NAME}:${IMAGE_TAG}
                """
            }
        }

        stage('Tag Docker Image') {
            steps {
                echo "Etiquetando imagen para Nexus: ${FULL_IMAGE}..."
                sh "docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${FULL_IMAGE}"
            }
        }

        stage('Push to Nexus') {
            steps {
                echo "Publicando imagen en Nexus..."
                withCredentials([usernamePassword(
                    credentialsId: CREDENTIALS_ID,
                    usernameVariable: 'NEXUS_USER',
                    passwordVariable: 'NEXUS_PASS'
                )]) {
                    sh """
                        echo \$NEXUS_PASS | docker login ${NEXUS_URL} \
                            --username \$NEXUS_USER \
                            --password-stdin
                        docker push ${FULL_IMAGE}
                        docker logout ${NEXUS_URL}
                    """
                }
            }
        }
    }

    post {
        always {
            echo "Limpiando imágenes locales..."
            sh """
                docker rmi ${IMAGE_NAME}:${IMAGE_TAG} || true
                docker rmi ${FULL_IMAGE}              || true
            """
        }
        success {
            echo "Pipeline completado exitosamente. Imagen publicada: ${FULL_IMAGE}"
        }
        failure {
            echo "Pipeline fallido. Revisar logs para más detalles."
        }
    }
}