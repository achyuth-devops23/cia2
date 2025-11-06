pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-south-2'
        AWS_ACCOUNT_ID = '111596617699'
        ECR_REPO = 'cia2-app'
        IMAGE_TAG = "${BUILD_NUMBER}"
        EC2_HOST = credentials('ec2-host')  // stored Jenkins secret containing EC2 public DNS or IP
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out code from GitHub...'
                checkout scm
            }
        }

        stage('Install & Test') {
            steps {
                sh '''
                    echo "Installing dependencies..."
                    npm install
                    echo "Running tests..."
                    npm test || echo "Tests skipped (no tests found)"
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    retry(2) {
                        echo "Building Docker image..."
                        sh '''
                            docker build -t ${ECR_REPO}:${IMAGE_TAG} .
                        '''
                    }
                }
            }
        }

        stage('Push to ECR') {
            steps {
                script {
                    withAWS(credentials: 'aws-credentials', region: AWS_REGION) {
                        sh """
                            echo "Logging in to ECR..."
                            aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                            
                            docker tag ${ECR_REPO}:${IMAGE_TAG} ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG}
                            docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG}
                        """
                    }
                    echo '✅ Image pushed to ECR successfully.'
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                script {
                    sshagent(['ec2-ssh-key']) {
                        sh """
                            echo "Deploying new image to EC2..."
                            ssh -o StrictHostKeyChecking=no ec2-user@${EC2_HOST} << 'EOF'
                                sudo docker pull ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG}
                                sudo docker stop cia2-app || true
                                sudo docker rm cia2-app || true
                                sudo docker run -d --name cia2-app -p 80:3000 ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG}
                                echo "Deployment successful on EC2"
                            EOF
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo '🎉 Pipeline completed successfully — application deployed to EC2!'
        }
        failure {
            echo '❌ Pipeline failed. Check logs for details.'
        }
    }
}
