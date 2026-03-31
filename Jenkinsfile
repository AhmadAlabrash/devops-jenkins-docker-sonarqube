pipeline {
    agent any

    environment {
        AWS_REGION         = 'eu-central-1'
        AWS_ACCOUNT_ID     = '612990353866'
        ECR_REPO           = 'my-nodejs-app'
        EKS_CLUSTER        = 'nodejs-lofi-walrus'
        APP_DIR            = 'Nodejs App'
        IMAGE_TAG          = "${BUILD_NUMBER}"
        IMAGE_URI          = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:${IMAGE_TAG}"
        IMAGE_LATEST       = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:latest"
        HELM_RELEASE       = 'my-nodejs-app'
        HELM_CHART_PATH    = './helm/my-nodejs-app'
        HELM_NAMESPACE     = 'default'
        SONAR_PROJECT_KEY  = 'my-nodejs-app'
        SONAR_PROJECT_NAME = 'my-nodejs-app'
    }

    stages {
        stage('Check Tools') {
            steps {
                sh '''
                    node -v
                    npm -v
                    docker --version
                    aws --version
                    kubectl version --client
                    helm version
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

        stage('SonarQube Scan') {
            steps {
                dir("${APP_DIR}") {
                    withSonarQubeEnv('sonarqube-server') {
                        sh '''
                            npx sonar-scanner \
                              -Dsonar.projectKey=$SONAR_PROJECT_KEY \
                              -Dsonar.projectName="$SONAR_PROJECT_NAME" \
                              -Dsonar.sources=src \
                              -Dsonar.tests=tests \
                              -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info
                        '''
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                dir("${APP_DIR}") {
                    sh """
                        docker build -t my-nodejs-app:${BUILD_NUMBER} .
                        docker tag my-nodejs-app:${BUILD_NUMBER} ${IMAGE_URI}
                        docker tag my-nodejs-app:${BUILD_NUMBER} ${IMAGE_LATEST}
                    """
                }
            }
        }

        stage('Push Image to ECR') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds'
                ]]) {
                    sh """
                        aws sts get-caller-identity

                        aws ecr get-login-password --region ${AWS_REGION} | \
                        docker login --username AWS --password-stdin \
                        ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

                        docker push ${IMAGE_URI}
                        docker push ${IMAGE_LATEST}
                    """
                }
            }
        }

        stage('Deploy to EKS with Helm') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: 'aws-creds'
                ]]) {
                    sh """
                        unset KUBECONFIG
                        unset KUBERNETES_MASTER
                        unset KUBE_MASTER
                        unset HELM_KUBEAPISERVER
                        unset HELM_KUBETOKEN
                        unset HELM_KUBECAFILE
                        unset HELM_KUBEINSECURE_SKIP_TLS_VERIFY

                        aws eks update-kubeconfig --region ${AWS_REGION} --name ${EKS_CLUSTER}

                        kubectl get nodes
                        helm lint ${HELM_CHART_PATH}

                        helm upgrade --install ${HELM_RELEASE} ${HELM_CHART_PATH} \
                          --namespace ${HELM_NAMESPACE} \
                          --create-namespace \
                          --set image.repository=${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO} \
                          --set image.tag=${BUILD_NUMBER} \
                          --kubeconfig /var/lib/jenkins/.kube/config

                        kubectl get pods -n ${HELM_NAMESPACE}
                        kubectl get svc -n ${HELM_NAMESPACE}
                        helm list -n ${HELM_NAMESPACE} --kubeconfig /var/lib/jenkins/.kube/config
                    """
                }
            }
        }
    }

    post {
        success {
            echo 'ECR push and Helm deploy to EKS done successfully'
        }
        failure {
            echo 'Pipeline failed'
        }
    }
}
