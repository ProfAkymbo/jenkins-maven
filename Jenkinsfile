pipeline {
    agent any

    tools {
        maven 'Maven3'
    }

    stages {
        stage('compile code') {
            steps {
                sh 'mvn compile'
            }
        }

        stage('PMD code-review') {
            steps {
                sh 'mvn -P metrics pmd:pmd'
            }
            post {
                success {
                    recordIssues(tools: [pmdParser(pattern: '**/pmd.xml')])
                }
            }
        }

        stage('Sonar Code Analysis') {
            environment {
                scannerHome = tool 'sonarqube-scanner'
            }
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh "${scannerHome}/bin/sonar-scanner"
                }
                timeout(time: 3, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('package app') {
            steps {
                sh 'mvn package'
            }
        }

        stage('publish app to jfrog') {
            steps {
                rtUpload (
                    serverId: 'jfrog-dev',
                    spec: '''{
                        "files": [
                            {
                                "pattern": "target/kitchensink.war",
                                "target": "non-prod-repo/"
                            }
                        ]
                    }'''
                )
            }
        }

        stage('Ansible Deploy to httpd') {
            steps {
                ansiblePlaybook(
                    credentialsId: 'ansible-token',
                    disableHostKeyChecking: true,
                    installation: 'ansible',
                    inventory: 'inventory',
                    playbook: 'playbook.yml'
                )
            }
        }

        stage('Docker Build & Run') {
            steps {
                sh '''
                docker ps | grep catalina | awk '{print $1}' | xargs -r docker stop || true
                docker build -t bloomy/myapp:1.0.${BUILD_NUMBER} .
                docker run -d -p 8050:8050 --name myapp-1.0.${BUILD_NUMBER} bloomy/myapp:1.0.${BUILD_NUMBER}
                '''
            }
        }
    }
}
