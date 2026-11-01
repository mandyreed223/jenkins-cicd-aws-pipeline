pipeline {
    agent {
        label 'docker'
    }

    environment {
        IMAGE_NAME = 'cloudops-app'
        TEST_CONTAINER = 'cloudops-test'

        AWS_REGION = 'us-east-1'
        ECR_REPOSITORY = 'cloudops-app'

        ECS_CLUSTER = 'cloudops-cluster'
        ECS_SERVICE = 'cloudops-service'
        ECS_TASK_FAMILY = 'cloudops-app'
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Source code checked out from GitHub.'
            }
        }

        stage('AWS Identity') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'aws-ecr-credentials',
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    sh '''
                        echo "Verifying Jenkins AWS identity..."
                        aws sts get-caller-identity
                    '''
                }
            }
        }

        stage('Build') {
            steps {
                sh '''
                    echo "Building Docker image..."
                    docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} .
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                    echo "Starting test container..."

                    docker rm -f ${TEST_CONTAINER} 2>/dev/null || true

                    docker run -d \
                        --name ${TEST_CONTAINER} \
                        ${IMAGE_NAME}:${BUILD_NUMBER}

                    sleep 2

                    docker exec ${TEST_CONTAINER} \
                        wget -qO- http://localhost \
                        | grep "Deployment Healthy"

                    echo "Application test passed."
                '''
            }
        }

        stage('ECR Push') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'aws-ecr-credentials',
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    sh '''
                        echo "Preparing image for Amazon ECR..."

                        AWS_ACCOUNT_ID=$(aws sts get-caller-identity \
                            --query Account \
                            --output text)

                        ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
                        ECR_IMAGE="${ECR_REGISTRY}/${ECR_REPOSITORY}:${BUILD_NUMBER}"

                        echo "Logging Docker into Amazon ECR..."

                        aws ecr get-login-password \
                            --region ${AWS_REGION} \
                            | docker login \
                                --username AWS \
                                --password-stdin ${ECR_REGISTRY}

                        echo "Tagging image..."

                        docker tag \
                            ${IMAGE_NAME}:${BUILD_NUMBER} \
                            ${ECR_IMAGE}

                        echo "Pushing image to Amazon ECR..."

                        docker push ${ECR_IMAGE}

                        echo "Image successfully pushed to Amazon ECR."
                    '''
                }
            }
        }

        stage('ECS Deploy') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'aws-ecr-credentials',
                        usernameVariable: 'AWS_ACCESS_KEY_ID',
                        passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                    )
                ]) {
                    sh '''
                        echo "Preparing ECS deployment..."

                        AWS_ACCOUNT_ID=$(aws sts get-caller-identity \
                            --query Account \
                            --output text)

                        ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
                        ECR_IMAGE="${ECR_REGISTRY}/${ECR_REPOSITORY}:${BUILD_NUMBER}"

                        echo "Reading current ECS task definition..."

                        aws ecs describe-task-definition \
                            --task-definition ${ECS_TASK_FAMILY} \
                            --region ${AWS_REGION} \
                            --query taskDefinition \
                            > current-task-definition.json

                        echo "Creating task definition for build ${BUILD_NUMBER}..."

                        ECR_IMAGE="${ECR_IMAGE}" python3 - <<'PY'
import json
import os

with open("current-task-definition.json") as f:
    task = json.load(f)

task["containerDefinitions"][0]["image"] = os.environ["ECR_IMAGE"]

allowed_fields = [
    "family",
    "taskRoleArn",
    "executionRoleArn",
    "networkMode",
    "containerDefinitions",
    "volumes",
    "placementConstraints",
    "requiresCompatibilities",
    "cpu",
    "memory",
    "pidMode",
    "ipcMode",
    "proxyConfiguration",
    "inferenceAccelerators",
    "ephemeralStorage",
    "runtimePlatform"
]

clean_task = {
    key: task[key]
    for key in allowed_fields
    if key in task
}

with open("new-task-definition.json", "w") as f:
    json.dump(clean_task, f, indent=2)
PY

                        echo "Registering new ECS task definition..."

                        NEW_TASK_DEFINITION=$(aws ecs register-task-definition \
                            --cli-input-json file://new-task-definition.json \
                            --region ${AWS_REGION} \
                            --query 'taskDefinition.taskDefinitionArn' \
                            --output text)

                        echo "New task definition registered."

                        echo "Updating ECS service..."

                        aws ecs update-service \
                            --cluster ${ECS_CLUSTER} \
                            --service ${ECS_SERVICE} \
                            --task-definition "${NEW_TASK_DEFINITION}" \
                            --region ${AWS_REGION} \
                            --query 'service.{Service:serviceName,TaskDefinition:taskDefinition,Desired:desiredCount}' \
                            --output table

                        echo "Waiting for ECS service to become stable..."

                        aws ecs wait services-stable \
                            --cluster ${ECS_CLUSTER} \
                            --services ${ECS_SERVICE} \
                            --region ${AWS_REGION}

                        echo "ECS deployment completed successfully."
                    '''
                }
            }
        }
    }

    post {
        always {
            sh '''
                echo "Cleaning up test container..."
                docker rm -f ${TEST_CONTAINER} 2>/dev/null || true
            '''
        }

        success {
            echo 'CI/CD pipeline completed successfully. 💚'
        }

        failure {
            echo 'CI/CD pipeline failed. Check the stage logs.'
        }
    }
}