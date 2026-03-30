pipeline {
    agent any

    environment {
        APP_DIR = 'Nodejs App'
        IMAGE_NAME = 'my-nodejs-app'
        IMAGE_TAG = "${BUILD_NUMBER}"
        FULL_IMAGE = "${IMAGE_NAME}:${IMAGE_TAG}"
    }

    triggers {
        //check repo every 5 minutes
        pollSCM('H/5 * * * *')
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/AhmadAlabrash/devops-jenkins-docker-sonarqube.git'
            }
        }

        stage('Show Files') {
            steps {
                sh '''
                    echo "Current workspace:"
                    pwd
                    ls -la
                    echo "Inside app folder:"
                    ls -la "$APP_DIR"
                '''
            }
        }

        stage('Install Dependencies') {
            steps {
                dir("${APP_DIR}") {
                    sh '''
                        npm install
                    '''
                }
            }
        }

        stage('Run Tests') {
            steps {
                dir("${APP_DIR}") {
                    sh '''
                        npm test
                    '''
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                dir("${APP_DIR}") {
                    sh '''
                        docker build -t $FULL_IMAGE .
                    '''
                }
            }
        }

        stage('Show Saved Image') {
            steps {
                sh '''
                    echo "Built image:"
                    docker images | grep $IMAGE_NAME || true
                    docker image inspect $FULL_IMAGE
                '''
            }
        }
    }

    post {
        success {
            echo "Pipeline finished successfully. Docker image saved as ${FULL_IMAGE}"
        }
        failure {
            echo "Pipeline failed."
        }
    }
}