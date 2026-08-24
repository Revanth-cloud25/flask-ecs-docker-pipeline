pipeline {
    agent any

    environment {
        AWS_ACCOUNT_ID = '343779419117'
        AWS_REGION = 'ap-south-1'
        ECR_REPO = 'flask-ecs-app'
        IMAGE_TAG = 'latest'
        ECR_URI = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}"
        ECS_CLUSTER = 'flask-ecs-cluster'
        ECS_SERVICE = 'flask-ecs-service'
        AWS_ACCESS_KEY_ID = credentials('aws-access-key-id')
        AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Docker Image') {
            steps {
                bat "docker build -t ${ECR_REPO}:${IMAGE_TAG} ."
            }
        }

        stage('Login to ECR') {
            steps {
                bat "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
            }
        }

        stage('Tag Image') {
            steps {
                bat "docker tag ${ECR_REPO}:${IMAGE_TAG} ${ECR_URI}:${IMAGE_TAG}"
            }
        }

        stage('Push to ECR') {
            steps {
                bat "docker push ${ECR_URI}:${IMAGE_TAG}"
            }
        }

        stage('Deploy to ECS') {
            steps {
                bat "aws ecs update-service --cluster ${ECS_CLUSTER} --service ${ECS_SERVICE} --force-new-deployment --region ${AWS_REGION}"
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully! New image deployed to ECS.'
        }
        failure {
            echo 'Pipeline failed. Check the stage logs above for details.'
        }
    }
}