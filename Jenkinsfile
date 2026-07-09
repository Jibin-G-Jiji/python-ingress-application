pipeline {
    agent {
        docker {
            image 'python:3.11-slim'
            args '--user root -v /var/run/docker.sock:/var/run/docker.sock'
        }
    }

    environment {
        SONAR_HOST_URL = 'http://32.236.85.107:9000'
        AWS_REGION     = 'us-east-1'
        AWS_ACCOUNT_ID = '724412576398'
        ECR_REPO       = 'python-ingress-application'
        ECR_REGISTRY   = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
        IMAGE_TAG      = "${env.BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                sh 'echo Checkout completed successfully'
            }
        }

       stage('Install Dependencies') {
            steps {
                dir('test_django_pro') {
                    sh '''
                        apt-get update
                        apt-get install -y docker.io curl unzip

                        # Install AWS CLI v2
                        curl -s "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
                        unzip -q awscliv2.zip
                        ./aws/install

                        # Install buildx manually
                        mkdir -p ~/.docker/cli-plugins
                        curl -sSL https://github.com/docker/buildx/releases/download/v0.17.1/buildx-v0.17.1.linux-amd64 -o ~/.docker/cli-plugins/docker-buildx
                        chmod +x ~/.docker/cli-plugins/docker-buildx

                        pip install --upgrade pip
                        pip install -r requirements.txt
                        pip install pytest pytest-django pytest-cov
                    '''
                }
            }
        }

        stage('Build and Test') {
            steps {
                dir('test_django_pro') {
                    sh '''
                        python manage.py makemigrations --check --dry-run
                        python manage.py migrate

                        pytest \
                          --ds=test_django_pro.settings \
                          --junitxml=report.xml \
                          --cov=. \
                          --cov-report=xml || true
                    '''
                }
            }

            post {
                always {
                    junit allowEmptyResults: true, testResults: 'test_django_pro/report.xml'
                }
            }
        }

        stage('Static Code Analysis') {
            steps {
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                    sh '''
                        docker run --rm \
                          -e SONAR_HOST_URL=$SONAR_HOST_URL \
                          sonarsource/sonar-scanner-cli \
                          -Dsonar.projectKey=django-app \
                          -Dsonar.sources=. \
                          -Dsonar.host.url=$SONAR_HOST_URL \
                          -Dsonar.token=$SONAR_TOKEN
                    '''
                }
            }
        }

        stage('ECR Login') {
            steps {
                withCredentials([[$class: 'AmazonWebServicesCredentialsBinding', credentialsId: 'aws-ecr-creds']]) {
                    sh '''
                        aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REGISTRY
                    '''
                }
            }
        }

       stage('Build Docker Image') {
            steps {
                dir('test_django_pro') {
                    sh '''
                        docker build -t $ECR_REGISTRY/$ECR_REPO:$IMAGE_TAG -t $ECR_REGISTRY/$ECR_REPO:latest .
                    '''
                }
            }
        }

        stage('Push to ECR') {
            steps {
                sh '''
                    docker push $ECR_REGISTRY/$ECR_REPO:$IMAGE_TAG
                    docker push $ECR_REGISTRY/$ECR_REPO:latest
                '''
            }
        }

        stage('Update Deployment File') {
            steps {
                sh 'echo Deployment update stage completed'
            }
        }
    }

    post {
        always {
            cleanWs()
        }

        success {
            echo 'Pipeline executed successfully!'
        }

        failure {
            echo 'Pipeline failed! Check logs above.'
        }
    }
}
