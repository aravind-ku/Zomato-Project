# Zomato-Project

pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node16'
    }
    environment {
        SCANNER_HOME = tool 'mysonar'
    }

    stages {
        stage('clean WS') {
            steps {
                cleanWs()
            }
        }
        stage('code') {
            steps {
                git url: 'https://github.com/aravind-ku/Zomato-Project.git', branch: 'master'
            }
        }
        stage('sonarqube Analysis') {
            steps {
                withSonarQubeEnv('mysonar') {
                    sh """
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectName=Zomato \
                        -Dsonar.projectKey=Zomato
                        """
                }
            }
        }
        stage('Quality Gate') {
            steps {
                script {
                    waitForQualityGate abortPipeline: false, credentialsId: 'sonar-password'
                }
            }
        }
        stage('install Dependencies') {
            steps {
                sh 'npm install'
            }
        }
        stage ('OWASP') {
            steps {
                dependencyCheck(
                    additionalArguments: '''
                        --scan ./ 
                        --disableYarnAudit 
                        --disableNodeAudit
                        --noupdate
                    ''',
                    odcInstallation: 'dp-check'
                )
        
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }
        stage('trivy') {
            steps {
                sh 'trivy fs . > trivyfs.txt'
            }
        }
        stage('Build') {
            steps {
                sh 'docker build -t image1 .'
            }
        }
        stage('Scan the image') {
            steps {
                sh 'trivy image image1'
            }
        }
        stage('container') {
            steps {
                sh 'docker run -d --name zom -p 3000:3000 image1'
            }
        }
    }
}
