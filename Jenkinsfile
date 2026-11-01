pipeline {
    agent {
        label 'docker'
    }

    environment {
        IMAGE_NAME = 'cloudops-app'
        TEST_CONTAINER = 'cloudops-test'

        AWS_REGION = 'us-east-1'
        ECR_REPOSITORY = 'cloudops-app'
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
    }

    post {
        always {
            sh '''
                echo "Cleaning up test container..."
                docker rm -f ${TEST_CONTAINER} 2>/dev/null || true
            '''
        }

        success {
            echo 'CI pipeline completed successfully. 💚'
        }

        failure {
            echo 'CI pipeline failed. Check the stage logs.'
        }
    }
}