pipeline {
    agent any

    environment {
        APP_NAME = 'catalogue'
        APP_VERSION = '1.0'
        IMAGE = 'ruthvikk1214/catalogue:1.0'
        NAMESPACE = 'roboshop'
    }

    options {
        timeout(time: 5, unit: 'MINUTES')
    }

    stages {
        // stage('SonarQube Analysis') {
        //     steps {
        //         withSonarQubeEnv('sonar-server') {
        //             sh '''
        //                 # Download sonar-scanner CLI if not present on the agent
        //                 if ! command -v sonar-scanner &> /dev/null; then
        //                     echo "Installing SonarQube Scanner CLI..."
        //                     curl -sSLo /tmp/sonar-scanner.zip https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-5.0.1.3006-linux.zip
        //                     unzip -q -o /tmp/sonar-scanner.zip -d /tmp/
        //                     export PATH="/tmp/sonar-scanner-5.0.1.3006-linux/bin:$PATH"
        //                 fi

        //                 sonar-scanner \
        //                 -Dsonar.projectKey=catalogue \
        //                 -Dsonar.projectName=catalogue \
        //                 -Dsonar.sources=.
        //             '''
        //         }
        //     }
        // }
        stage('Pull Docker Image') {
            steps {
                sh '''
                    echo "Pulling $IMAGE"
                    docker pull $IMAGE
                '''
            }
        }

        stage('Scan Image with Trivy') {
            steps {
                sh '''
                    echo "Scanning $IMAGE"

                    trivy image \
                    --severity HIGH,CRITICAL \
                    --exit-code 1 \
                    $IMAGE || true
                '''
            }
        }

        stage('Create K8s Deployment') {
            steps {
                sh '''
                    echo "Deploying $APP_NAME"

                    # Authenticate and point kubectl to the EKS cluster
                    aws eks update-kubeconfig --name roboshop --region us-east-1

                    kubectl apply \
                    -f manifest.yaml \
                    -n $NAMESPACE
                '''
            }
        }

        stage('Verify Catalogue Deployment') {
            steps {
                sh '''
                    echo "Verifying $APP_NAME deployment"

                    kubectl rollout status \
                    deployment/$APP_NAME \
                    -n $NAMESPACE \
                    --timeout=120s

                    kubectl get pods \
                    -n $NAMESPACE \
                    -l component=$APP_NAME
                '''
            }
        }
    }
}