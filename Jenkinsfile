pipeline {
    agent any

    environment {
        AWS_REGION   = 'eu-central-1'
        AWS_ACCOUNT_ID = '612990353866'
        ECR_REPO     = 'my-nodejs-app'
        EKS_CLUSTER  = 'nodejs-lofi-walrus'
        APP_DIR      = 'Nodejs App'
        IMAGE_TAG    = "${BUILD_NUMBER}"
        IMAGE_URI    = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG}"
        IMAGE_LATEST = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:latest"
        DEPLOYMENT_NAME = 'my-nodejs-app'
        CONTAINER_NAME  = 'my-nodejs-app'
    }

    triggers {
        pollSCM('H/5 * * * *')
    }

    stages {
        stage('Show Tools') {
            steps {
                sh '''
                    node -v
                    npm -v
                    docker --version
                    aws --version
                    kubectl version --client
                '''
            }
        }

        stage('Install Dependencies') {
            steps {
                dir("${APP_DIR}") {
                    sh 'npm install'
                }
            }
        }

        stage('Run Tests') {
            steps {
                dir("${APP_DIR}") {
                    sh 'npm test'
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                dir("${APP_DIR}") {
                    sh '''
                        docker build -t my-nodejs-app:${BUILD_NUMBER} .
                        docker tag my-nodejs-app:${BUILD_NUMBER} ${IMAGE_URI}
                        docker tag my-nodejs-app:${BUILD_NUMBER} ${IMAGE_LATEST}
                    '''
                }
            }
        }

stage('Login to ECR') {
    steps {
        withCredentials([[
            $class: 'AmazonWebServicesCredentialsBinding',
            credentialsId: 'aws-creds'
        ]]) {
            sh '''
                aws ecr get-login-password --region ${AWS_REGION} | \
                docker login --username AWS --password-stdin \
                ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
            '''
        }
    }
}

        stage('Push Image to ECR') {
            steps {
                sh '''
                    docker push ${IMAGE_URI}
                    docker push ${IMAGE_LATEST}
                '''
            }
        }

        stage('Connect to EKS') {
            steps {
                sh '''
                    aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER}
                    kubectl get nodes
                '''
            }
        }

        stage('Deploy to EKS') {
            steps {
                sh '''
                    kubectl get deployment ${DEPLOYMENT_NAME} >/dev/null 2>&1 || kubectl create deployment ${DEPLOYMENT_NAME} --image=${IMAGE_URI}

                    kubectl set image deployment/${DEPLOYMENT_NAME} ${CONTAINER_NAME}=${IMAGE_URI} --record || true
                    kubectl expose deployment ${DEPLOYMENT_NAME} --type=LoadBalancer --port=80 --target-port=3000 || true

                    kubectl rollout status deployment/${DEPLOYMENT_NAME}
                    kubectl get pods
                    kubectl get svc
                '''
            }
        }
    }

    post {
        success {
            echo "Done: image pushed to ECR and deployed to EKS"
        }
        failure {
            echo "Pipeline failed"
        }
    }
}
