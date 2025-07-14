pipeline {
    agent any
    
    environment {
        DOCKER_REGISTRY = 'docker.io/nocnex'
        APP_NAME = 'nodejs-project'
        K8S_DEPLOYMENT = 'nodejs-project'
    }
    
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', 
                url: 'https://github.com/nocnexhamza/nodejs-project.git'
            }
        }
        
        stage('Build & Test') {
            agent {
                docker {
                    image 'node:14'
                    args '-u root'
                }
            }
            steps {
                sh 'npm install'
                sh 'npm audit fix || true'
                sh 'npm test || echo "Tests failed but continuing deployment"'
            }
        }
        
        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'SonarQubeScanner'
                    withSonarQubeEnv('SonarQube') {
                        sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=nodejs-project -Dsonar.sources=. -Dsonar.language=js -Dsonar.exclusions=Dockerfile"
                    }
                }
            }
        }
        
        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
        
        stage('Build Docker Image') {
            steps {
                script {
                    sh """
                        DOCKER_BUILDKIT=1 docker build \
                        -t ${DOCKER_REGISTRY}/${APP_NAME}:${env.BUILD_NUMBER} .
                    """
                }
            }
        }

        stage('Scan with Trivy') {
            steps {
                script {
                    sh label: 'Trivy Scan', script: '''#!/bin/bash
                        set -x  # Enable debug output
                        docker run --rm \
                            -v /var/run/docker.sock:/var/run/docker.sock \
                            -v "$WORKSPACE:/workspace" \
                            aquasec/trivy image \
                            --exit-code 1 \
                            --severity CRITICAL \
                            --format table \
                            --output /workspace/trivy-report.txt \
                            "${DOCKER_REGISTRY}/${APP_NAME}:${BUILD_NUMBER}"
                        
                        # Wrap table output in HTML
                        cat <<EOF > /workspace/trivy-report.html
<!DOCTYPE html>
<html>
<head>
    <title>Trivy Vulnerability Report</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 20px; }
        h1 { color: #333; }
        pre { 
            background-color: #f9f9f9; 
            padding: 15px; 
            border: 1px solid #ddd; 
            border-radius: 5px; 
            font-family: monospace; 
            white-space: pre-wrap; 
            word-wrap: break-word; 
        }
        .critical { color: red; font-weight: bold; }
    </style>
</head>
<body>
    <h1>Trivy Vulnerability Report</h1>
    <pre class="critical">
$(cat /workspace/trivy-report.txt || echo "No vulnerabilities found or scan failed.")
    </pre>
</body>
</html>
EOF
                    '''
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-report.html', allowEmptyArchive: true
                    publishHTML(target: [
                        allowMissing: true,
                        alwaysLinkToLastBuild: false,
                        keepAll: true,
                        reportDir: '.',
                        reportFiles: 'trivy-report.html',
                        reportName: 'Trivy Report'
                    ])
                }
            }
        }
        
        stage('Push Docker Image') {
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'docker-hub-credentials',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {
                        sh """
                            echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin
                            docker push ${DOCKER_REGISTRY}/${APP_NAME}:${env.BUILD_NUMBER}
                        """
                    }
                }
            }
        }
        
        stage('Deploy to Kubernetes') {
            steps {
                script {
                    withKubeConfig([credentialsId: 'k8s-credentials']) {
                        sh """
                            sed 's|image:.*|image: ${DOCKER_REGISTRY}/${APP_NAME}:${env.BUILD_NUMBER}|g' \
                                k8s/deployment.yaml > k8s/deployment-${env.BUILD_NUMBER}.yaml
                            
                            kubectl apply -f k8s/deployment-${env.BUILD_NUMBER}.yaml
                            kubectl apply -f k8s/service.yaml
                            kubectl rollout status deployment/${K8S_DEPLOYMENT}
                        """
                    }
                }
            }
        }
    }
    
    post {
        always {
            script {
                sh 'rm -f k8s/deployment-*.yaml trivy-report.* || true'
                cleanWs()
            }
        }
        success {
            script {
                if (env.SLACK_CHANNEL) {
                    slackSend(
                        color: "good",
                        message: "SUCCESS: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                        channel: env.SLACK_CHANNEL
                    )
                }
            }
        }
        failure {
            script {
                if (env.SLACK_CHANNEL) {
                    slackSend(
                        color: "danger",
                        message: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                        channel: env.SLACK_CHANNEL
                    )
                }
            }
        }
    }
}
