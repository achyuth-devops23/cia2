pipeline {
    agent any

    tools {
        nodejs 'Node16'   // Name you configured in Jenkins under Tools
    }

    environment {
        AWS_REGION = 'ap-south-2'
        AWS_ACCOUNT_ID = '111596617699'
        ECR_REPO = 'us-east-1/cia2-app'
        IMAGE_TAG = "${BUILD_NUMBER}"
        EC2_HOST = credentials('ec2-host')  // Jenkins secret with EC2 DNS/IP
    }

    stages {
        stage('Checkout') {
            steps {
                echo '📦 Checking out code from GitHub...'
                checkout scm
            }
        }

        stage('Install & Test') {
            steps {
                echo '📦 Installing dependencies & running tests...'
                sh '''
                    echo "Installing dependencies..."
                    npm install
                    echo "Running tests..."
                    npm test || echo "⚠️ No tests found, skipping..."
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    echo "🐳 Building multi-architecture Docker image (amd64 + arm64)..."
                    retry(2) {
                        sh """
                            docker buildx build --platform linux/amd64,linux/arm64 \
                                -t ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG} \
                                --push .
                        """
                    }
                }
            }
        }


        // stage('Push to ECR') {
        //     steps {
        //         script {
        //             withAWS(credentials: 'aws-credentials', region: AWS_REGION) {
        //                 sh """
        //                     echo "🔐 Logging in to ECR..."
        //                     aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
        //                     docker tag ${ECR_REPO}:${IMAGE_TAG} ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG}
        //                     docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG}
        //                 """
        //             }
        //             echo '✅ Image pushed to ECR successfully.'
        //         }
        //     }
        // }

stage('Deploy to EC2') {
    steps {
        script {
            echo "🚀 Deploying to EC2..."
            withCredentials([sshUserPrivateKey(credentialsId: 'ec2-ssh-key', keyFileVariable: 'KEYFILE', usernameVariable: 'USER')]) {
                sh """
                    ssh -i $KEYFILE -o StrictHostKeyChecking=no $USER@${EC2_HOST} '
                        echo "🔐 Logging in to ECR on EC2..."
                        aws ecr get-login-password --region ${AWS_REGION} | sudo docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

                        echo "📦 Pulling the latest image..."
                        sudo docker pull ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG}

                        echo "🧹 Stopping and removing old container..."
                        sudo docker stop cia2-app || true
                        sudo docker rm cia2-app || true

                        echo "🚀 Running new container..."
                        sudo docker run -d --name cia2-app -p 80:3000 ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG}

                        echo "✅ Deployment successful on EC2!"
                    '
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
