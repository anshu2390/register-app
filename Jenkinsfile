pipeline {
    agent {
        kubernetes {
            yaml """
apiVersion: v1
kind: Pod
spec:
    containers:
    - name: dind
      image: docker:24.0.5-dind
      securityContext:
        privileged: true
      volumeMounts:
      - name: docker-graph-storage
        mountPath: /var/lib/docker
    - name: jnlp
      image: jenkins/inbound-agent:alpine
    volumes:
    - name: docker-graph-storage
      emptyDir: {}
"""
        }
    }
    tools {
        jdk 'Java17'
        maven 'Maven3'
    }
    environment {
        APP_NAME = "register-app"
        RELEASE = "1.0.0"
        IMAGE_NAME = "anshu2390/${APP_NAME}"
        IMAGE_TAG = "${RELEASE}-${BUILD_NUMBER}" 
    }
    stages{
        stage("Cleanup Workspace"){
            steps {
                cleanWs()
            }
        }
        stage("Checkout GitHub Repo"){
            steps {
                git branch: 'main', credentialsId: 'github', url: 'https://github.com/anshu2390/register-app'
            }
        }
        stage("Build Application"){
            steps {
                sh "mvn clean package"
            }
        }
        stage("Test Application"){
            steps {
                sh "mvn test"
            }
        }
        stage("Sonarqube Analysis"){
            steps {
                script {
                    withSonarQubeEnv(credentialsId: 'jenkins-sonarqube-token') {
                        sh "mvn sonar:sonar"
                    }
                }
            }
        }
        stage("Quality Gate"){
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'jenkins-sonarqube-token'
                }
            }
        }
        stage("Build and Push Docker Image"){
            steps {
                container('dind') {
                    sh 'docker --version'
                    withCredentials([usernamePassword(credentialsId: 'dockerhub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh 'echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin'
                        sh 'docker build -t "$IMAGE_NAME:$IMAGE_TAG" .'
                        sh 'docker tag "$IMAGE_NAME:$IMAGE_TAG" "$IMAGE_NAME:latest"'
                        sh 'docker push "$IMAGE_NAME:$IMAGE_TAG"'
                        sh 'docker push "$IMAGE_NAME:latest"'
                    }
                }
            }
        }
        stage("Trivy Scan"){
            steps {
                script {
                    sh ('docker run -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image anshu2390/register-app:latest --no-progress --scanners vuln --exit-code 0 --severity HIGH,CRITICAL --format table')
                }
            }
        }
        stage("Cleanup Artifacts"){
            steps {
                script {
                    sh "docker rmi $IMAGE_NAME:$IMAGE_TAG"
                    sh "docker rmi $IMAGE_NAME:latest"
                }
            }
        }
    }
}
