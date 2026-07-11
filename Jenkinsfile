pipeline {
    agent any

    environment {
        AWS_REGION      = 'ap-south-1'
        ECR_ACCOUNT_ID  = '562904760755'
        ECR_REGISTRY    = "${ECR_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        HELLO_REPO      = 'streamingapp-helloservice'
        PROFILE_REPO    = 'streamingapp-profileservice'
        FRONTEND_REPO   = 'streamingapp-frontend'
        IMAGE_TAG       = "${env.BUILD_NUMBER}"
        MONGO_URL       = credentials('mongo-url')
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'git@github.com:shinmaheshwari/streamingapp-devops-platform.git', credentialsId: 'github-ssh'
            }
        }

        stage('Build Images') {
            steps {
                sh "docker build -t ${HELLO_REPO}:${IMAGE_TAG} ./helloService"
                sh "docker build -t ${PROFILE_REPO}:${IMAGE_TAG} ./profileService"
                sh "docker build -t ${FRONTEND_REPO}:${IMAGE_TAG} ./frontend"
            }
        }

        stage('Login to ECR') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
                    sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}"
                }
            }
        }

        stage('Push Images to ECR') {
            steps {
                sh """
                    docker tag ${HELLO_REPO}:${IMAGE_TAG} ${ECR_REGISTRY}/${HELLO_REPO}:${IMAGE_TAG}
                    docker tag ${PROFILE_REPO}:${IMAGE_TAG} ${ECR_REGISTRY}/${PROFILE_REPO}:${IMAGE_TAG}
                    docker tag ${FRONTEND_REPO}:${IMAGE_TAG} ${ECR_REGISTRY}/${FRONTEND_REPO}:${IMAGE_TAG}

                    docker push ${ECR_REGISTRY}/${HELLO_REPO}:${IMAGE_TAG}
                    docker push ${ECR_REGISTRY}/${PROFILE_REPO}:${IMAGE_TAG}
                    docker push ${ECR_REGISTRY}/${FRONTEND_REPO}:${IMAGE_TAG}
                """
            }
        }
    }

    post {
        success {
            echo "Build #${BUILD_NUMBER} succeeded and pushed to ECR."
        }
        failure {
            echo "Build #${BUILD_NUMBER} failed."
        }
    }
}
