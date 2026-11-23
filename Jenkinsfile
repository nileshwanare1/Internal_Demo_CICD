
pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-south-1'
        AWS_ACCOUNT_ID = '601510715785'

        ECR_REPOSITORY = 'static-website'
        IMAGE_TAG = "${BUILD_NUMBER}"

        ECR_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPOSITORY}"

        ECS_CLUSTER = 'ECS_Cluster'
        ECS_SERVICE = 'static-website-task-service'
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

                        def taskDefinitionArn = sh(
                    script: '''
                        set -e

                        aws ecs describe-task-definition \
                        --task-definition static-website-task \
                        --region ap-south-1 \
                        --query taskDefinition \
                        --output json > task-definition-current.json

                        jq -e --arg name nginx-webapp \
                        '.containerDefinitions[] | select(.name == $name)' \
                        task-definition-current.json > /dev/null

                        jq --arg container nginx-webapp \
                        --arg image 601510715785.dkr.ecr.ap-south-1.amazonaws.com/static-website:${BUILD_NUMBER} \
                        '
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

                        aws ecs register-task-definition \
                        --cli-input-json file://task-definition-new.json \
                        --region ap-south-1 \
                        --query 'taskDefinition.taskDefinitionArn' \
                        --output text
                    ''',
                    returnStdout: true
                ).trim()

                if (!taskDefinitionArn.startsWith('arn:aws:ecs:')) {
                    error("Invalid ECS Task Definition ARN: ${taskDefinitionArn}")
                }

                env.NEW_TASK_DEFINITION = taskDefinitionArn

                echo "New Task Definition: ${env.NEW_TASK_DEFINITION}"
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

    echo "Updating ECS service..."

    aws ecs update-service \
    --cluster "${ECS_CLUSTER}" \
    --service "${ECS_SERVICE}" \
    --task-definition "${NEW_TASK_DEFINITION}" \
    --region "${AWS_REGION}"

    echo "Waiting for ECS service stability..."

    aws ecs wait services-stable \
    --cluster "${ECS_CLUSTER}" \
    --services "${ECS_SERVICE}" \
    --region "${AWS_REGION}"

    echo "ECS deployment completed successfully!"
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
