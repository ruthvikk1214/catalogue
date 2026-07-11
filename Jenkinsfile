pipeline {
    agent any {
        node {
            label 'linux'
        }
    }
    options {
        //disableConcurrentBuilds()
        timeout(time: 5, unit: 'MINUTES')
    }
    stages {
        stage('Read Version') {
            steps {
                script {
                    def packageJson = readJSON file: 'package.json'
                    appName = packageJson.name
                    appVersion = packageJson.version
                    echo "Building ${appName} version ${appVersion}"
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                sh '''
                    
                '''
            }
        }
        stage('Build Docker Image') {
            steps {
                sh """
                    echo "Building Docker image"
                    docker build -t ${appName}:${appVersion} .
                """
            }
        }
        stage
    }
}