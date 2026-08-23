pipeline {
    agent any

    environment {
        APP_NAME        = 'catalogue'
        AWS_REGION      = 'us-east-1'
        AWS_ACCOUNT_ID  = 'YOUR_AWS_ACCOUNT_ID' // Replace with your AWS Account ID or Jenkins Credential
        NAMESPACE       = 'roboshop'
    }

    options {
        timeout(time: 10, unit: 'MINUTES')
    }

    stages {
        stage('Read version') {
            steps {
                script {
                    def packageJson = readJSON file: 'package.json' 
                    env.APP_VERSION = packageJson.version
                    env.FULL_IMAGE  = "${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.AWS_REGION}.amazonaws.com/${env.APP_NAME}:${env.APP_VERSION}"
                    echo "Application version: ${env.APP_VERSION}"
                    echo "Target Image: ${env.FULL_IMAGE}"
                }
            }
        }

       stage('Build & Push Docker Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'aws-creds', 
                    usernameVariable: 'AWS_ACCESS_KEY_ID', 
                    passwordVariable: 'AWS_SECRET_ACCESS_KEY'
                )]) {
                    sh '''
                        aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
                        docker build -t ${FULL_IMAGE} .
                        docker push ${FULL_IMAGE}
                    '''
                }
            }
        }

        stage('Scan Image with Trivy') {
            steps {
                sh '''
                    echo "Scanning ${FULL_IMAGE}"
                    trivy image --severity HIGH,CRITICAL --exit-code 1 ${FULL_IMAGE} || true
                '''
            }
        }

        stage('Create K8s Deployment') {
            steps {
                sh '''
                    echo "Deploying ${APP_NAME}"
                    aws eks update-kubeconfig --name roboshop --region ${AWS_REGION}
                    kubectl apply -f manifest.yaml -n ${NAMESPACE}
                '''
            }
        }

        stage('Verify Catalogue Deployment') {
            steps {
                sh '''
                    echo "Verifying ${APP_NAME} deployment"
                    kubectl rollout status deployment/${APP_NAME} -n ${NAMESPACE} --timeout=120s
                    kubectl get pods -n ${NAMESPACE} -l component=${APP_NAME}
                '''
            }
        }
    }
}