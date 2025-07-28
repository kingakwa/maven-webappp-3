// Jenkinsfile for Maven Web Application CI/CD Pipeline

pipeline {
    agent any

    environment {
        // SonarQube credentials (created in Jenkins)
        SONAR_CRED = credentials('sonarqube-token-id') // Replace with your actual Jenkins credential ID for SonarQube token
        // Nexus credentials (created in Jenkins)
        NEXUS_CRED_ID = 'nexus-creds' // Replace with your actual Jenkins credential ID for Nexus username/password
        DOCKERHUB_CRED_ID = 'dockerhub-creds' // Replace with your actual Jenkins credential ID for Docker Hub username/password
        APP_NAME = "maven-webapp"
        DOCKER_IMAGE_NAME = "kingakwa/${APP_NAME}" // Replace kingakwa with your Docker Hub username
        NEXUS_URL = "http://${NEXUS_SERVER_PRIVATE_IP}:8081" // Replace with your Nexus private IP
        SONARQUBE_URL = "http://${SONARQUBE_SERVER_PRIVATE_IP}:9000" // Replace with your SonarQube private IP
        // Monitoring Server IP for Trivy scanning
        MONITORING_SERVER_IP = "${MONITORING_SERVER_PRIVATE_IP}" // Replace with your Monitoring server private IP
    }

    stages {
        stage('Source Code Management') {
            steps {
                echo "Cloning repository: ${env.GIT_URL}"
                git branch: 'main', url: 'https://github.com/kingakwa/maven-webappp-3.git'
            }
        }

        stage('Run Unit Tests') {
            steps {
                echo "Running Maven unit tests..."
                // Assuming pom.xml is in the root of the cloned repository
                sh "mvn test"
            }
        }

        stage('Quality Code Assurance - SonarQube') {
            steps {
                echo "Performing SonarQube analysis..."
                withSonarQubeEnv(credentialsId: env.SONAR_CRED) {
                    sh "mvn clean verify sonar:sonar -Dsonar.projectKey=${APP_NAME} -Dsonar.host.url=${SONARQUBE_URL}"
                }
            }
            post {
                always {
                    script {
                        // Gate check for SonarQube analysis
                        // Wait for quality gate status
                        timeout(time: 5, unit: 'MINUTES') {
                            def qg = waitForQualityGate()
                            if (qg.status != 'OK') {
                                error "SonarQube Quality Gate failed with status: ${qg.status}"
                            } else {
                                echo "SonarQube Quality Gate passed."
                            }
                        }
                    }
                }
            }
        }

        stage('Vulnerability Scan - Trivy (Code)') {
            steps {
                echo "Running Trivy filesystem scan on the source code..."
                // Assume Trivy is installed on the Jenkins agent
                // Or, if Trivy is on the monitoring server, we'd need to SSH and execute
                // For simplicity, let's assume Trivy is installed on the Jenkins server (or agent)
                sh "trivy fs --format json -o trivy-code-scan.json ."
                // Optional: Add a step to parse and fail the build if critical vulnerabilities are found
                // sh "trivy fs --severity CRITICAL,HIGH --exit-code 1 ." // Fails if critical/high are found
            }
        }

        stage('Build Application') {
            steps {
                echo "Building Maven application..."
                sh "mvn clean package -DskipTests" // Skip tests as they were run in a previous stage
            }
        }

        stage('Artifacts Upload - Nexus') {
            steps {
                echo "Uploading artifacts to Nexus..."
                withCredentials([usernamePassword(credentialsId: env.NEXUS_CRED_ID, usernameVariable: 'NEXUS_USERNAME', passwordVariable: 'NEXUS_PASSWORD')]) {
                    // Deploy to Nexus releases or snapshots based on versioning logic
                    // For simplicity, deploying to a generic 'releases' repo
                    sh "mvn deploy -s settings.xml -DaltDeploymentRepository=nexus-releases::default::${NEXUS_URL}/repository/maven-releases/"
                    // You might need a settings.xml file in your project or Jenkins global settings
                    // Example settings.xml for Nexus deploy:
                    /*
                    <settings>
                        <servers>
                            <server>
                                <id>nexus-releases</id>
                                <username>${env.NEXUS_USERNAME}</username>
                                <password>${env.NEXUS_PASSWORD}</password>
                            </server>
                            <server>
                                <id>nexus-snapshots</id>
                                <username>${env.NEXUS_USERNAME}</username>
                                <password>${env.NEXUS_PASSWORD}</password>
                            </server>
                        </servers>
                    </settings>
                    */
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                echo "Building Docker image..."
                script {
                    // Assuming Dockerfile is in the root of the project
                    docker.build("${DOCKER_IMAGE_NAME}:${BUILD_NUMBER}").push()
                    docker.build("${DOCKER_IMAGE_NAME}:latest").push() // Also push as latest
                }
            }
        }

        stage('Scan Docker Image - Trivy') {
            steps {
                echo "Running Trivy image scan..."
                // Assume Trivy is installed on the Jenkins agent
                // If Trivy is on the monitoring server, you'd need to SCP the image or pull it there and scan.
                // For now, assuming Jenkins agent can run Trivy on the locally built image.
                sh "trivy image --format json -o trivy-image-scan.json ${DOCKER_IMAGE_NAME}:${BUILD_NUMBER}"
                // sh "trivy image --severity CRITICAL,HIGH --exit-code 1 ${DOCKER_IMAGE_NAME}:${BUILD_NUMBER}" // Fails if critical/high are found
            }
        }

        stage('Push Docker Image to DockerHub') {
            steps {
                echo "Pushing Docker image to DockerHub..."
                script {
                    withDockerRegistry(credentialsId: env.DOCKERHUB_CRED_ID, url: 'https://index.docker.io/v1/') {
                        docker.image("${DOCKER_IMAGE_NAME}:${BUILD_NUMBER}").push()
                        docker.image("${DOCKER_IMAGE_NAME}:latest").push()
                    }
                }
            }
        }

        stage('Deployment to Kubernetes') {
            steps {
                echo "Deploying to Kubernetes..."
                // This stage requires a running Kubernetes cluster and kubectl configured on Jenkins.
                // For a full production setup, you would use a more robust deployment strategy (e.g., Helm, ArgoCD).
                // This is a placeholder for a basic kubectl apply.

                // Replace with actual Kubernetes deployment commands
                // Example:
                // sh "kubectl apply -f kubernetes/deployment.yaml"
                // sh "kubectl set image deployment/${APP_NAME} ${APP_NAME}=${DOCKER_IMAGE_NAME}:${BUILD_NUMBER}"
                echo "Kubernetes deployment steps go here. (e.g., kubectl apply -f deployment.yaml)"
            }
        }

        stage('Security - Kubernetes Scan (Optional)') {
            steps {
                echo "Running security checks on Kubernetes deployment (e.g., Kube-bench, Aqua Security tools)..."
                // Placeholder for Kubernetes security scanning tools post-deployment
                echo "Kubernetes security scanning steps."
            }
        }

        stage('Email Notification') {
            steps {
                echo "Sending email notification..."
                mail bcc: '', body: "Pipeline build ${currentBuild.displayName} for ${env.JOB_NAME} is ${currentBuild.result}!", cc: '', from: 'jenkins@example.com', replyTo: '', subject: "Jenkins Build Notification: ${env.JOB_NAME} - ${currentBuild.result}", to: 'your-email@example.com' // Replace with actual email
            }
        }

        stage('Monitoring Application') {
            steps {
                echo "Monitoring application status with Prometheus and Grafana..."
                // This is a conceptual stage. Actual monitoring is continuous.
                // Here, you might trigger specific alerts or check health endpoints.
                echo "Application is being monitored through Prometheus and Grafana."
            }
        }
    }

    post {
        success {
            echo "Pipeline completed successfully!"
        }
        failure {
            echo "Pipeline failed! Check logs for details."
            // You might add additional notification steps here, e.g., Slack
        }
        always {
            echo "Pipeline finished."
        }
    }
}
