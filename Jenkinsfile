pipeline {
    agent any

    environment {
        IMAGE_NAME = "demo-cloud-app:latest"
        CONTAINER_PORT = "8000"
        HOST_PORT = "1000"
        TRIVY_EXIT_CODE = 0
    }

    stages {
        stage('Build Docker Image') {
            steps {
                echo 'Building Docker image...'
                sh "docker build -t ${IMAGE_NAME} ."
            }
        }

        stage('Run Trivy Scan from Docker') {
            steps {
                script {
                    echo 'Scanning Docker image for vulnerabilities using Trivy (Docker)...'

                    TRIVY_EXIT_CODE = sh(
                        script: """
                            docker run --rm \
                            -v trivy_data:/root/.cache/ \
                            -v /var/run/docker.sock:/var/run/docker.sock \
                            aquasec/trivy image --exit-code 1 --severity CRITICAL ${IMAGE_NAME} || true
                        """,
                        returnStatus: true
                    )

                    echo "Trivy scan exit code: ${TRIVY_EXIT_CODE}"

                    if (TRIVY_EXIT_CODE == 1) {
                        error "Critical vulnerabilities detected. Blocking deployment."
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    sh "docker rm -f ${JOB_NAME} || true"

                    sh """
                        docker run \
                            -d \
                            --restart always \
                            -p ${HOST_PORT}:${CONTAINER_PORT} \
                            --name ${JOB_NAME} \
                            ${IMAGE_NAME}
                    """
                }
            }
        }

        stage('Cloudflared Tunnel') {
            steps {
                script {
                    configureTunnel()
                }
            }
        }

    }

    post {
        failure {
            script {
                if (TRIVY_EXIT_CODE == 1) {
                    echo "Pipeline failed due to detected critical vulnerabilities."
                    // Send an email notification on Trivy failure only
                    emailext(
                        subject: "Deployment Blocked: Critical Vulnerability Detected in Docker Image",
                        body: "The deployment pipeline was halted due to critical vulnerabilities found in the Docker image. Please review the Trivy scan report.",
                        to: "rahulsood707@gmail.com",
                        attachLog: true
                    )
                }
            }
        }
    }
}
