pipeline {
    agent any

    environment {
        IMAGE_NAME = "nileshwanare/myapp:${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                url: 'https://github.com/nileshwanare1/Internal_Demo_CICD.git'
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                sh 'mvn sonar:sonar'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${IMAGE_NAME} ."
            }
        }

        stage('Push Docker Image') {
            steps {
                sh "docker push ${IMAGE_NAME}"
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh 'kubectl apply -f deployment.yaml'
                sh 'kubectl apply -f service.yaml'
            }
        }

        stage('Health Check') {
            steps {
                sh 'curl -f http://application-url/actuator/health'
            }
        }

        stage('Monitoring Validation') {
            steps {
                echo 'Validating application metrics'

                // Prometheus metrics endpoint
                sh 'curl http://application-url:8080/actuator/prometheus'

                // Grafana dashboard reachable
                sh 'curl http://grafana-server:3000/api/health'

                // CloudWatch verification
                sh 'aws cloudwatch list-metrics'

                echo 'Monitoring validation successful'
            }
        }
    }

    post {
        success {
            echo 'Deployment Successful'
        }

        failure {
            echo 'Deployment Failed - Trigger Rollback'

            sh '''
               kubectl rollout undo deployment/myapp
            '''
        }
    }
}