
pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-south-1'
        AWS_ACCOUNT_ID = '601510715785'

        ECR_REPOSITORY = 'static-website'
        IMAGE_TAG = "${BUILD_NUMBER}"

        ECR_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY}"

        ECS_CLUSTER = 'static-website-cluster'
        ECS_SERVICE = 'static-website-service'
        ECS_TASK_FAMILY = 'static-website-task'
        ECS_CONTAINER_NAME = 'nginx-webapp'

        AWS_DEFAULT_REGION = 'ap-south-1'
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

        // NEW STAGE 1: Register ECS Task Definition

        stage('Register ECS Task Definition') {

            steps {

                script {

                    withCredentials([
                        usernamePassword(
                            credentialsId: 'aws_new',
                            usernameVariable: 'AWS_ACCESS_KEY_ID',
                            passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                        )
                    ]) {

                        env.TASK_DEF_ARN = sh(
                            returnStdout: true,
                            script: '''
                                set -e

                                export AWS_ACCESS_KEY_ID
                                export AWS_SECRET_ACCESS_KEY

                                echo "Fetching existing ECS task definition..."

                                aws ecs describe-task-definition \
                                    --task-definition ${ECS_TASK_FAMILY} \
                                    --region ${AWS_REGION} \
                                    --query 'taskDefinition' \
                                    --output json > task-definition-current.json

                                echo "Validating container name..."

                                jq -e --arg name "${ECS_CONTAINER_NAME}" \
                                    '.containerDefinitions[] | select(.name == $name)' \
                                    task-definition-current.json > /dev/null

                                echo "Updating container image..."

                                jq --arg container "${ECS_CONTAINER_NAME}" \
                                   --arg image "${ECR_URI}:${IMAGE_TAG}" '

                                    del(
                                        .taskDefinitionArn,
                                        .revision,
                                        .status,
                                        .requiresAttributes,
                                        .compatibilities,
                                        .registeredAt,
                                        .registeredBy,
                                        .deregisteredAt
                                    )
                                    |
                                    (
                                        .containerDefinitions[]
                                        | select(.name == $container)
                                        | .image
                                    ) = $image

                                ' task-definition-current.json > task-definition-new.json

                                echo "Registering new ECS task definition..."

                                aws ecs register-task-definition \
                                    --cli-input-json file://task-definition-new.json \
                                    --region ${AWS_REGION} \
                                    --query 'taskDefinition.taskDefinitionArn' \
                                    --output text
                            '''
                        ).trim()

                        echo "New Task Definition: ${env.TASK_DEF_ARN}"
                    }
                }
            }
        }

        // NEW STAGE 2: Deploy to ECS Fargate

        stage('Deploy to ECS Fargate') {

            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'aws_new',
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {

                    sh '''
                        set -e

                        export AWS_ACCESS_KEY_ID
                        export AWS_SECRET_ACCESS_KEY

                        echo "Updating ECS service..."

                        aws ecs update-service \
                            --cluster ${ECS_CLUSTER} \
                            --service ${ECS_SERVICE} \
                            --task-definition ${TASK_DEF_ARN} \
                            --region ${AWS_REGION}

                        echo "Waiting for ECS deployment to complete..."

                        aws ecs wait services-stable \
                            --cluster ${ECS_CLUSTER} \
                            --services ${ECS_SERVICE} \
                            --region ${AWS_REGION}

                        echo "ECS deployment completed successfully!"

                        echo "Checking ECS service status..."

                        aws ecs describe-services \
                            --cluster ${ECS_CLUSTER} \
                            --services ${ECS_SERVICE} \
                            --region ${AWS_REGION} \
                            --query 'services[0].[status,desiredCount,runningCount,pendingCount]' \
                            --output table
                    '''
                }
            }
        }
    }

    post {
        success {
            echo 'CI/CD Pipeline Completed Successfully!'
            echo 'Docker Image Pushed to ECR.'
            echo 'New ECS Task Definition Registered.'
            echo 'Application Deployed to ECS Fargate.'
            echo "ECR Image: ${ECR_URI}:${IMAGE_TAG}"
        }

        failure {
            echo 'Pipeline Failed - Check Jenkins Console Output.'
        }
    }
}
