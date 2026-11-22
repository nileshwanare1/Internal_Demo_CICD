
pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-south-1'

        AWS_ACCOUNT_ID = '601510715785'

        ECR_REPOSITORY = 'static-website'

        ECR_REGISTRY = "${601510715785}.dkr.ecr.${ap-south-1}.amazonaws.com"

        IMAGE_TAG = "${BUILD_NUMBER}"

        ECR_IMAGE = "${ECR_REGISTRY}/${ECR_REPOSITORY}:${IMAGE_TAG}"
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
        stage('Docker Image Build') {
            steps {
                sh '''
                    docker build -t ${ECR_IMAGE} .
                '''

                echo 'Docker image created successfully'
            }
        }

        // Stage 3: Verify AWS credentials and login to ECR
        stage('AWS ECR Authentication') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'aws-ecr-creds',
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    sh '''
                        set +x

                        export AWS_DEFAULT_REGION=${AWS_REGION}

                        aws sts get-caller-identity

                        aws ecr get-login-password \
                        --region ${AWS_REGION} |
                        docker login \
                        --username AWS \
                        --password-stdin ${ECR_REGISTRY}
                    '''
                }

                echo 'AWS ECR authentication successful'
            }
        }

        // Stage 4: Push Docker image to ECR
        stage('Push Docker Image to ECR') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'aws-ecr-creds',
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    sh '''
                        set +x

                        docker push ${ECR_IMAGE}
                    '''
                }

                echo 'Docker image pushed to ECR successfully'
            }
        }

        // Stage 5: Verify Docker image
        stage('Verify ECR Image') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'aws-ecr-creds',
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    sh '''
                        export AWS_DEFAULT_REGION=${AWS_REGION}

                        aws ecr describe-images \
                        --repository-name ${ECR_REPOSITORY} \
                        --region ${AWS_REGION}
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'CI/CD Pipeline Completed Successfully!'
            echo 'Docker image is available in AWS ECR.'
        }

        failure {
            echo 'CI/CD Pipeline Failed. Check Console Output.'
        }
    }
}
