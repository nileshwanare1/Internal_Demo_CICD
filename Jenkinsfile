
pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-south-1'
        AWS_ACCOUNT_ID = '601510715785'
        ECR_REPOSITORY = 'static-website'
        IMAGE_TAG = '1'
    }

    stages {

        // Stage 1: Checkout GitHub repository
        stage('GitHub Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/nileshwanare1/Internal_Demo_CICD.git'

                echo 'GitHub checkout successful'
            }
        }

        // Stage 2: Build Docker image
        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t nginx-webapp:${IMAGE_TAG} .
                '''
            }
        }

        // Stage 3: Stop old container
        stage('Stop Old Container') {
            steps {
                sh '''
                    docker rm -f nginx-webapp || true
                '''
            }
        }

        // Stage 4: Run new container
        stage('Run New Container') {
            steps {
                sh '''
                    docker run -d \
                    --name nginx-webapp \
                    -p 82:80 \
                    nginx-webapp:${IMAGE_TAG}
                '''
            }
        }
    }

    post {
        success {
            echo 'CI/CD Pipeline Completed Successfully!'
            echo 'Docker container is running on port 82.'
        }

        failure {
            echo 'CI/CD Pipeline Failed. Check Console Output.'
        }
    }
}
