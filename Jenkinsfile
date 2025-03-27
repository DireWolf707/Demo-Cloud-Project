pipeline {
    agent any

    environment {
        IMAGE_NAME = "demo-cloud-app:latest"
        CONTAINER_PORT = "8000"
        HOST_PORT = "1000"
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

                    // Run Trivy inside a Docker container and generate JSON output
                    sh """
                        docker run --rm \
                        aquasec/trivy image ${IMAGE_NAME}
                    """

                    // Parse the JSON to count critical vulnerabilities using jq
                    def criticalCount = sh(
                        script: "jq '[.Results[].Vulnerabilities[]? | select(.Severity==\"CRITICAL\")] | length' trivy-result.json",
                        returnStdout: true
                    ).trim().toInteger()

                    echo "Critical vulnerabilities found: ${criticalCount}"

                    // If critical vulnerabilities are found, fail the pipeline
                    if (criticalCount >= 1) {
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
            echo "Pipeline failed due to detected critical vulnerabilities."
            // Send an email notification on failure
            emailext(
                subject: "Deployment Blocked: Critical Vulnerability Detected in Docker Image",
                body: "The deployment pipeline was halted due to critical vulnerabilities found in the Docker image. Please review the Trivy scan report attached for more details.",
                to: "rahulsood707@gmail.com",
                attachLog: true
            )
        }
    }
}
