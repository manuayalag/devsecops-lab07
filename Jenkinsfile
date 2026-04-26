pipeline {
    agent any

    options {
        timeout(time: 10, unit: 'MINUTES')
    }

    environment {
        // Nombre interno de la imagen Docker
        IMAGE_NAME = "sumador"
        IMAGE_TAG  = "${env.BUILD_NUMBER}"

        // Credencial almacenada en Jenkins (usuario/contraseña de Nexus)
        // NUNCA hardcodear credenciales en el código
        CREDENTIALS_ID = "nexus-credentials"

        // Nexus corre en el mismo host Docker, accedido por nombre de servicio
        // "localhost" NO funciona entre contenedores; usar el nombre del servicio
        NEXUS_HOST = "nexus:8083"
        NEXUS_URL  = "http://nexus:8083"
        NEXUS_REPO = "docker-hosted"

        // Nombre completo de la imagen en Nexus
        FULL_IMAGE = "${NEXUS_HOST}/${IMAGE_NAME}:${IMAGE_TAG}"
    }

    stages {

        // ─────────────────────────────────────────────
        // 1. Checkout del código desde GitHub
        // ─────────────────────────────────────────────
        stage('Checkout') {
            steps {
                checkout scm
                echo "Código descargado. Build #${IMAGE_TAG}"
            }
        }

        // ─────────────────────────────────────────────
        // 2. Análisis de dependencias (npm audit)
        //    Shift-Left: detectar vulnerabilidades
        //    ANTES de construir la imagen
        // ─────────────────────────────────────────────
        stage('Dependency Scan - npm audit') {
            steps {
                echo "Analizando dependencias con npm audit..."
                sh "docker run --rm ${IMAGE_NAME}:${IMAGE_TAG} sh -c 'npm audit --audit-level=critical || true'"
            }
        }

        // ─────────────────────────────────────────────
        // 3. Build de la imagen Docker
        // ─────────────────────────────────────────────
        stage('Build Docker Image') {
            steps {
                echo "Construyendo imagen Docker ${IMAGE_NAME}:${IMAGE_TAG}..."
                sh "docker build -t ${IMAGE_NAME}:${IMAGE_TAG} ."
            }
        }

        // ─────────────────────────────────────────────
        // 4. Ejecución de tests unitarios
        // ─────────────────────────────────────────────
        stage('Run Tests') {
            steps {
                echo "Ejecutando tests unitarios..."
                sh "docker run --rm ${IMAGE_NAME}:${IMAGE_TAG} npm test"
            }
        }

        // ─────────────────────────────────────────────
        // 5. Escaneo de vulnerabilidades con Trivy
        //    Falla el pipeline si hay CVEs CRÍTICOS
        // ─────────────────────────────────────────────
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

        // ─────────────────────────────────────────────
        // 6. Etiquetado de la imagen para Nexus
        // ─────────────────────────────────────────────
        stage('Tag Docker Image') {
            steps {
                echo "Etiquetando imagen para Nexus: ${FULL_IMAGE}..."
                sh "docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${FULL_IMAGE}"
            }
        }

        // ─────────────────────────────────────────────
        // 7. Publicación en Nexus
        //    Credenciales gestionadas por Jenkins
        //    (no hardcodeadas en el pipeline)
        // ─────────────────────────────────────────────
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
