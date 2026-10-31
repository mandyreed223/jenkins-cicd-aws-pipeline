pipeline {
    agent {
        label 'docker'
    }

    environment {
        IMAGE_NAME = 'cloudops-app'
        TEST_CONTAINER = 'cloudops-test'
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Source code checked out from GitHub.'
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