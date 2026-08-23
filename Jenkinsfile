pipeline {
    agent any

    environment {
        APP_NAME       = 'catalogue'
        AWS_REGION     = 'us-east-1'
        AWS_ACCOUNT_ID = 'YOUR_AWS_ACCOUNT_ID' // replace with real ID or use a Jenkins credential
        NAMESPACE      = 'roboshop'
    }

    options {
        timeout(time: 10, unit: 'MINUTES')
    }

    stages {
        stage('Read version') {
            steps {
                script {
                    def pkg = readJSON file: 'package.json'
                    env.APP_VERSION = pkg.version
                    env.FULL_IMAGE = "${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.AWS_REGION}.amazonaws.com/${env.APP_NAME}:${env.APP_VERSION}"
                    echo "Application version: ${env.APP_VERSION}"
                    echo "Target image: ${env.FULL_IMAGE}"
                }
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                script {
                    // Use Jenkins AWS Credentials binding
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                     credentialsId: 'aws-creds',
                                     accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                                     secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                        // Configure AWS CLI for the given region
                        sh "aws configure set aws_access_key_id $AWS_ACCESS_KEY_ID"
                        sh "aws configure set aws_secret_access_key $AWS_SECRET_ACCESS_KEY"
                        sh "aws configure set default.region $AWS_REGION"

                        // Login to ECR
                        sh '''
                            aws ecr get-login-password --region $AWS_REGION |
                                docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.$AWS_REGION.amazonaws.com
                        '''

                        // Build, tag and push the image
                        sh '''
                            docker build -t ${FULL_IMAGE} .
                            docker push ${FULL_IMAGE}
                        '''
                    }
                }
            }
        }

        stage('Scan Image with Trivy') {
            steps {
                sh '''
                    echo "Scanning ${FULL_IMAGE}"
                    trivy image --severity HIGH,CRITICAL --exit-code 1 ${FULL_IMAGE}
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

    post {
        always {
            sh '''
                docker rmi ${FULL_IMAGE} || true
            '''
            cleanWs()
        }
        success { echo "✅ Pipeline completed successfully." }
        failure { echo "❌ Pipeline failed. Check console output for details." }
    }
}