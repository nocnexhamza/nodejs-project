pipeline {
    agent {
        kubernetes {
            label 'jenkins-agent'
            yaml """
apiVersion: v1
kind: Pod
metadata:
  name: jenkins-agent
spec:
  containers:
  - name: jnlp
    image: nocnex/jenkins-agent:nerdctlv4
    resources:
      requests:
        memory: "512Mi"
        cpu: "500m"
    securityContext:
      privileged: true
    volumeMounts:
    - mountPath: "/home/jenkins/agent"
      name: "workspace-volume"
  - name: node
    image: node:18-slim
    command: ["cat"]
    tty: true
    resources:
      requests:
        memory: "256Mi"
        cpu: "100m"
    volumeMounts:
    - mountPath: "/home/jenkins/agent"
      name: "workspace-volume"
  - name: kubectl
    image: bitnami/kubectl:1.29
    command: ["cat"]
    tty: true
    resources:
      requests:
        memory: "256Mi"
        cpu: "100m"
    volumeMounts:
    - mountPath: "/home/jenkins/agent"
      name: "workspace-volume"
  - name: dind
    image: docker:dind
    securityContext:
      privileged: true
    resources:
      requests:
        memory: "512Mi"
        cpu: "500m"
    volumeMounts:
    - mountPath: "/var/run/docker.sock"
      name: "docker-sock"
    - mountPath: "/home/jenkins/agent"
      name: "workspace-volume"
    env:
    - name: DOCKER_TLS_CERTDIR
      value: ""
  volumes:
  - name: "workspace-volume"
    emptyDir: {}
  - name: "docker-sock"
    emptyDir: {}
"""
        }
    }

    environment {
        DOCKER_IMAGE = "nocnex/nodejs-app-v2"
        DOCKER_TAG = "${env.BUILD_NUMBER}"
        GIT_REPO = "https://github.com/nocnexhamza/nodejs-project.git"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Wait for Docker') {
            steps {
                container('dind') {
                    script {
                        // Increased timeout and added health check
                        retry(5) {
                            timeout(time: 2, unit: 'MINUTES') {
                                sh '''
                                    while ! docker ps; do
                                        echo "Waiting for Docker to be ready..."
                                        sleep 5
                                    done
                                    docker info
                                '''
                            }
                        }
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                container('dind') {
                    script {
                        docker.withRegistry('https://registry.hub.docker.com', 'dockerhublogin') {
                            docker.build("${env.DOCKER_IMAGE}:${env.DOCKER_TAG}").push()
                        }
                    }
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                container('kubectl') {
                    script {
                        def deploymentYaml = """
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodejs-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nodejs-app
  template:
    metadata:
      labels:
        app: nodejs-app
    spec:
      containers:
      - name: nodejs-container
        image: ${env.DOCKER_IMAGE}:${env.DOCKER_TAG}
        ports:
        - containerPort: 3000
        resources:
          limits:
            cpu: "1000m"
            memory: "512Mi"
          requests:
            cpu: "500m"
            memory: "256Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: nodejs-service
spec:
  type: ClusterIP
  selector:
    app: nodejs-app
  ports:
  - port: 80
    targetPort: 3000
"""
                        writeFile file: 'deployment.yaml', text: deploymentYaml

                        withCredentials([file(credentialsId: 'kubernetes_file', variable: 'KUBECONFIG')]) {
                            sh """
                                export KUBECONFIG=\${KUBECONFIG}
                                kubectl apply -f deployment.yaml
                            """
                        }
                    }
                }
            }
        }

        stage('Verify Deployment') {
            steps {
                container('kubectl') {
                    withCredentials([file(credentialsId: 'kubernetes_file', variable: 'KUBECONFIG')]) {
                        sh """
                            export KUBECONFIG=\${KUBECONFIG}
                            kubectl rollout status deployment/nodejs-app --timeout=300s
                            kubectl get pods,svc
                        """
                    }
                }
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
