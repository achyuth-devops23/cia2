pipeline {
    agent any
    
    environment {
        AWS_REGION = 'ap-south-2'
        ECR_REPO = 'my-web-app'
        AWS_ACCOUNT_ID = credentials('111596617699')
        IMAGE_TAG = "${BUILD_NUMBER}"
        EC2_HOST = credentials('ec2-host')
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                echo 'Code checked out successfully'
            }
        }
        
        stage('Build') {
            steps {
                sh 'npm install'
                echo 'Application built successfully'
            }
        }
        
        stage('Test') {
            steps {
                sh 'npm test'
                echo 'Tests passed'
            }
        }
        
        stage('Docker Build') {
            steps {
                script {
                    sh "docker build -t ${ECR_REPO}:${IMAGE_TAG} ."
                    echo "Docker image built: ${ECR_REPO}:${IMAGE_TAG}"
                }
            }
        }
        
        stage('Push to ECR') {
            steps {
                script {
                    withAWS(credentials: 'aws-credentials', region: AWS_REGION) {
                        sh """
                            aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                            docker tag ${ECR_REPO}:${IMAGE_TAG} ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG}
                            docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG}
                        """
                    }
                    echo 'Image pushed to ECR'
                }
            }
        }
        
        stage('Deploy to EC2') {
            steps {
                script {
                    sshagent(['ec2-ssh-key']) {
                        sh """
                            ssh -o StrictHostKeyChecking=no ec2-user@${EC2_HOST} 'bash -s' < deploy-remote.sh ${IMAGE_TAG}
                        """
                    }
                    echo 'Deployed to EC2'
                }
            }
        }
    }
    
    post {
        success {
            echo 'Pipeline completed successfully!'
        }
        failure {
            echo 'Pipeline failed!'
        }
    }
}
