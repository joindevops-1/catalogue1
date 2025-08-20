pipeline {
    agent  {
        label 'AGENT-1'
    }
    environment { 
        appVersion = ''
        REGION = "us-east-1"
        ACC_ID = "315069654700"
        PROJECT = "roboshop"
        COMPONENT = "catalogue"
    }
    options {
        timeout(time: 30, unit: 'MINUTES') 
        disableConcurrentBuilds()
    }
    parameters {
        booleanParam(name: 'deploy', defaultValue: false, description: 'Toggle this value')
    }
    // Build
    stages {
        stage('Read package.json') {
            steps {
                script {
                    def packageJson = readJSON file: 'package.json'
                    appVersion = packageJson.version
                    echo "Package version: ${appVersion}"
                }
            }
        }
        stage('Install Dependencies') {
            steps {
                script {
                   sh """
                        npm install
                   """
                }
            }
        }
        stage('Unit Testing') {
            steps {
                script {
                   sh """
                        echo "unit tests"
                   """
                }
            }
        }
        /* stage('Sonar Scan') {
            environment {
                scannerHome = tool 'sonar-7.2'
            }
            steps {
                script {
                   // Sonar Server envrionment
                   withSonarQubeEnv(installationName: 'sonar-7.2') {
                         sh "${scannerHome}/bin/sonar-scanner"
                   }
                }
            }
        }
        stage("Quality Gate") {
            steps {
              timeout(time: 1, unit: 'HOURS') {
                waitForQualityGate abortPipeline: true
              }
            }
        } */
        stage('Scan Dependabot Alerts') {
            when {
                not {
                    branch 'main'
                }
            }
            environment {
                GITHUB_TOKEN = credentials('github-token') // Store your GitHub token in Jenkins credentials           // Replace with your GitHub repo
            }

            steps {
                script {
                    def response = sh(
                        script: """
                            curl -s -H "Authorization: token ${GITHUB_TOKEN}" \\
                            -H "Accept: application/vnd.github+json" \\
                            https://api.github.com/repos/DAWS-82S/shipping/dependabot/alerts
                        """,
                        returnStdout: true
                    )

                    def alerts = readJSON text: response
                    def highOrCritical = alerts.findAll { 
                        it.security_vulnerability?.severity in ['high', 'critical']
                    }

                    if (highOrCritical.size() > 0) {
                        echo "Found ${highOrCritical.size()} high/critical vulnerabilities!"
                        error("Failing build due to unresolved security issues.")
                    } else {
                        echo "No high or critical vulnerabilities found."
                    }
                }
            }
        }


        stage('Docker Build') {
            steps {
                script {
                    withAWS(credentials: 'aws-creds', region: 'us-east-1') {
                        sh """
                            aws ecr get-login-password --region ${REGION} | docker login --username AWS --password-stdin ${ACC_ID}.dkr.ecr.us-east-1.amazonaws.com
                            docker build -t ${ACC_ID}.dkr.ecr.us-east-1.amazonaws.com/${PROJECT}/${COMPONENT}:${appVersion} .
                            docker push ${ACC_ID}.dkr.ecr.us-east-1.amazonaws.com/${PROJECT}/${COMPONENT}:${appVersion}
                        """
                    }
                }
            }
        }
        stage('Trigger Deploy') {
            when{
                expression { params.deploy }
            }
            steps {
                script {
                    build job: 'catalogue-cd',
                    parameters: [
                        string(name: 'appVersion', value: "${appVersion}"),
                        string(name: 'deploy_to', value: 'dev')
                    ],
                    propagate: false,  // even SG fails VPC will not be effected
                    wait: false // VPC will not wait for SG pipeline completion
                }
            }
        }
        
    }

    post { 
        always { 
            echo 'I will always say Hello again!'
            deleteDir()
        }
        success { 
            echo 'Hello Success'
        }
        failure { 
            echo 'Hello Failure'
        }
    }
}