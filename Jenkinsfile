pipeline {
    agent any

    environment {
        APP_NAME       = 'catalogue'
        ECR_REPO       = 'roboshop/catalogue'
        ECR_REGION     = 'ap-south-1'
        EKS_REGION     = 'us-east-1'
        AWS_ACCOUNT_ID = '628087992516'
        NAMESPACE      = 'roboshop'
    }

    options {
        timeout(time: 10, unit: 'MINUTES')
    }

    stages {
        stage('Read version') {
            steps {
                script {
                    // Append build number to prevent ECR immutable tag push errors
                    // Use sh to extract version — avoids needing the Pipeline Utility Steps plugin
                    def version = sh(
                        script: "node -p \"require('./package.json').version\" 2>/dev/null || grep -m1 '\"version\"' package.json | sed 's/.*: *\"//;s/\".*//'",
                        returnStdout: true
                    ).trim()
                    env.APP_VERSION = "${version}-${BUILD_NUMBER}"
                    env.FULL_IMAGE  = "${env.AWS_ACCOUNT_ID}.dkr.ecr.${env.ECR_REGION}.amazonaws.com/${env.ECR_REPO}:${env.APP_VERSION}"
                    echo "Application version: ${env.APP_VERSION}"
                    echo "Target image: ${env.FULL_IMAGE}"
                }
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                script {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                     credentialsId: 'aws-creds',
                                     accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                                     secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                        sh '''
                            # Log in to ECR
                            aws ecr get-login-password --region ${ECR_REGION} | \
                                docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${ECR_REGION}.amazonaws.com

                            # Build and push the image
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
                    if ! command -v trivy &> /dev/null; then
                        echo "Trivy not found, installing..."
                        # Install wget if needed
                        sudo dnf -y install wget || true
                        wget -qO - https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sudo sh -s -- -b /usr/local/bin
                    fi
                    trivy image --severity HIGH,CRITICAL --exit-code 1 ${FULL_IMAGE} || true
                '''
            }
        }

        stage('Create K8s Deployment') {
            steps {
                script {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                     credentialsId: 'aws-creds',
                                     accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                                     secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                        sh '''
                            echo "Connecting to EKS cluster in ${EKS_REGION}..."
                            aws eks update-kubeconfig --name roboshop --region ${EKS_REGION}

                            # Update manifest with new image tag
                            sed -i "s|image:.*|image: ${FULL_IMAGE}|g" manifest.yaml

                            # Apply Kubernetes manifests
                            kubectl apply -f manifest.yaml -n ${NAMESPACE} --validate=false
                        '''
                    }
                }
            }
        }

        stage('Verify Catalogue Deployment') {
            steps {
                script {
                    withCredentials([[$class: 'AmazonWebServicesCredentialsBinding',
                                     credentialsId: 'aws-creds',
                                     accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                                     secretKeyVariable: 'AWS_SECRET_ACCESS_KEY']]) {
                        sh '''
                            echo "Verifying ${APP_NAME} deployment..."
                            kubectl rollout status deployment/${APP_NAME} -n ${NAMESPACE} --timeout=120s
                            kubectl get pods -n ${NAMESPACE} -l component=${APP_NAME}
                        '''
                    }
                }
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