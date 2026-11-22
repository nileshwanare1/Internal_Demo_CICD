
pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-south-1'
        AWS_ACCOUNT_ID = '601510715785'
        ECR_REPOSITORY = 'static-website'
        IMAGE_TAG = "${BUILD_NUMBER}"

        ECR_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY}"
    }

    stages {

        stage('GitHub Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/nileshwanare1/Internal_Demo_CICD.git'

                echo 'GitHub checkout successful'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build -t nginx-webapp:${IMAGE_TAG} .
                '''
            }
        }

        stage('AWS ECR Login') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'aws_new',
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    sh '''
                        export AWS_ACCESS_KEY_ID
                        export AWS_SECRET_ACCESS_KEY

                        aws sts get-caller-identity

                        aws ecr get-login-password \
                        --region ${AWS_REGION} | \
                        docker login \
                        --username AWS \
                        --password-stdin \
                        ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                    '''
                }
            }
        }

        stage('Tag Docker Image') {
            steps {
                sh '''
                    docker tag nginx-webapp:${IMAGE_TAG} \
                    ${ECR_URI}:${IMAGE_TAG}

                    docker tag nginx-webapp:${IMAGE_TAG} \
                    ${ECR_URI}:latest
                '''
            }
        }

        stage('Push Image to ECR') {
            steps {
                sh '''
                    docker push ${ECR_URI}:${IMAGE_TAG}
                    docker push ${ECR_URI}:latest
                '''
            }
        }

        stage('Deploy Local Container') {
            steps {
                sh '''
                    docker rm -f nginx-webapp || true

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
            echo 'Docker Image Successfully Pushed To ECR'
            echo "ECR Image: ${ECR_URI}:${IMAGE_TAG}"
        }

        failure {
            echo 'Pipeline Failed'
        }
    }
}
