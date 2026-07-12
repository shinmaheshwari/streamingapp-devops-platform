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
        SNS_TOPIC_ARN   = "arn:aws:sns:${AWS_REGION}:${ECR_ACCOUNT_ID}:streamingapp-deploy-notifications"
    }

    stages {

        stage('Checkout') {
            steps {
                retry(conditions: [nonresumable()], count: 2) {
                    git branch: 'main',
                        url: 'git@github.com:shinmaheshwari/streamingapp-devops-platform.git',
                        credentialsId: 'github-creds'
                }
            }
        }

        stage('Login to ECR') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
                    sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}"
                }
            }
        }

        stage('Build & Push Images') {
            steps {
                sh """
                    docker buildx build --platform linux/amd64 --provenance=false --sbom=false \
                      -t ${ECR_REGISTRY}/${HELLO_REPO}:${IMAGE_TAG} \
                      -t ${ECR_REGISTRY}/${HELLO_REPO}:latest \
                      --push ./backend/helloService
                """
                sh """
                    docker buildx build --platform linux/amd64 --provenance=false --sbom=false \
                      -t ${ECR_REGISTRY}/${PROFILE_REPO}:${IMAGE_TAG} \
                      -t ${ECR_REGISTRY}/${PROFILE_REPO}:latest \
                      --push ./backend/profileService
                """
                sh """
                    docker buildx build --platform linux/amd64 --provenance=false --sbom=false \
                      -t ${ECR_REGISTRY}/${FRONTEND_REPO}:${IMAGE_TAG} \
                      -t ${ECR_REGISTRY}/${FRONTEND_REPO}:latest \
                      --push ./frontend
                """
            }
        }

        stage('Deploy to EKS') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
                    sh "aws eks update-kubeconfig --name streamingapp-cluster --region ${AWS_REGION}"
                    sh """
                        helm upgrade --install streamingapp ./streamingapp-chart \
                          --set helloService.tag=${IMAGE_TAG} \
                          --set profileService.tag=${IMAGE_TAG} \
                          --set frontend.tag=${IMAGE_TAG}
                    """
                }
            }
        }
    }

    post {
        success {
            withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
                sh """
                    aws sns publish --topic-arn ${SNS_TOPIC_ARN} \
                      --message 'Deployment SUCCESS: Build #${BUILD_NUMBER}' --region ${AWS_REGION}
                """
            }
            echo "Build #${BUILD_NUMBER} succeeded — images pushed to ECR and deployed to EKS."
        }
        failure {
            withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-creds']]) {
                sh """
                    aws sns publish --topic-arn ${SNS_TOPIC_ARN} \
                      --message 'Deployment FAILED: Build #${BUILD_NUMBER}' --region ${AWS_REGION}
                """
            }
            echo "Build #${BUILD_NUMBER} failed."
        }
    }
}

