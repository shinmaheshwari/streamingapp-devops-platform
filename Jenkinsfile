pipeline {

    agent any

    environment {
        AWS_REGION='ap-south-1'
        ACCOUNT_ID='562904760755'
    }

    stages {

        stage('Checkout') {
            steps {
                git 'git@github.com:shinmaheshwari/streamingapp-devops-platform.git'
            }
        }

        stage('Build Docker Images') {
            steps {
                sh 'docker build -t hello-service backend/helloService'
                sh 'docker build -t profile-service backend/profileService'
                sh 'docker build -t streaming-frontend frontend'
            }
        }

        stage('Push To ECR') {
            steps {

                sh '''
                aws ecr get-login-password \
                --region $AWS_REGION \
                | docker login \
                --username AWS \
                --password-stdin \
                $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
                '''
            }
        }
    }
}
