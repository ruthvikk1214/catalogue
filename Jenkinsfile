pipeline {
    agent any

    environment {
        APP_NAME = ''
        APP_VERSION = ''
    }

    options {
        timeout(time: 5, unit: 'MINUTES')
    }

    stages {
        stage('Pull Docker Image') {
            steps {
                script {
                    def packageJson = readJSON file: 'package.json'

                    env.APP_NAME = packageJson.name
                    env.APP_VERSION = packageJson.version

                    echo "Pulling ${env.APP_NAME} version ${env.APP_VERSION}"
                }

                sh '''
                    docker pull ruthvikk1214/catalogue:$APP_VERSION
                '''
            }
        }

        stage('Scan Image with Trivy') {
            steps {
                sh '''
                    trivy image \
                    --severity HIGH,CRITICAL \
                    --exit-code 1 \
                    ruthvikk1214/catalogue:$APP_VERSION
                '''
            }
        }

        stage('Create K8s Deployment') {
            steps {
                sh '''
                    kubectl apply -f manifest.yaml -n roboshop
                '''
            }
        }

        stage('Verify Catalogue Deployment') {
            steps {
                sh '''
                    kubectl rollout status deployment/catalogue \
                    -n roboshop \
                    --timeout=120s

                    kubectl get pods \
                    -n roboshop \
                    -l component=catalogue
                '''
            }
        }
    }
}