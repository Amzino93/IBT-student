pipeline {
    agent any
    options {
        timeout(time: 20, unit: 'MINUTES')
    }
    stages{
        // NPM dependencies
        stage('pull npm dependencies') {
            steps {
                sh 'cd app && npm install'
            }
        }
        // Run Unit test
        stage('Run Unit Test') {
            steps {
                sh 'cd app && npm test'
            }
        }
        // run sonarqube test
        stage('Run Sonarqube') {
            environment {
                scannerHome = tool 'ibt-sonarqube';
            }
            steps {
              withSonarQubeEnv(credentialsId: 'SQ-student', installationName: 'IBT sonarqube') {
                sh "${scannerHome}/bin/sonar-scanner"
              }
            }
        }
        stage('build Docker Container') {
            steps {
                script {
                    // build image
                    docker.build("630437092685.dkr.ecr.us-east-2.amazonaws.com/ibt-student:latest")
                }
            }
        }
        stage('Push to ECR') {
            steps {
                script{
                    //https://<AwsAccountNumber>.dkr.ecr.<region>.amazonaws.com/ibt-student', 'ecr:<region>:<credentialsId>
                    docker.withRegistry('https://630437092685.dkr.ecr.us-east-2.amazonaws.com/ibt-student', 'ecr:us-east-2:ibt-ecr') {
                    // build image
                    def myImage = docker.build("630437092685.dkr.ecr.us-east-2.amazonaws.com/ibt-student:latest")
                    // push image
                    myImage.push()
                    }
                }
            }
        }
        stage('Trivy Scan (Aqua)') {
            steps {
                sh 'trivy image --format template --template "@/var/lib/jenkins/trivy_tmp/html.tpl" --output trivy_report.html 630437092685.dkr.ecr.us-east-2.amazonaws.com/ibt-student:latest'
            }
        }
        stage('Deploy to dev') {
            steps {
                withCredentials([[
                    $class: 'AmazonWebServicesCredentialsBinding',
                    credentialsId: "amza-ecr",
                    accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                    secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                ]]){
                    ansiblePlaybook(
                          playbook: 'ansible/deploy-docker.yaml',
                          inventory: 'ansible/hosts',
                          credentialsId: 'amza',
                          colorized: true,
                          extraVars: [
                              "myHosts" : "devServer",
                              "compose_file": "${WORKSPACE}/docker-compose.yaml",
                              "access_key": AWS_ACCESS_KEY_ID,
                              "access_secret": AWS_SECRET_ACCESS_KEY
                          ]
                      )
                }
            }
        }
        stage('Deploy to prod') {
                steps {
                    withCredentials([[
                        $class: 'AmazonWebServicesCredentialsBinding',
                        credentialsId: "amza-ecr",
                        accessKeyVariable: 'AWS_ACCESS_KEY_ID',
                        secretKeyVariable: 'AWS_SECRET_ACCESS_KEY'
                    ]]){
                        ansiblePlaybook(
                              playbook: 'ansible/deploy-docker.yaml',
                              inventory: 'ansible/hosts',
                              credentialsId: 'amza',
                              colorized: true,
                              extraVars: [
                                  "myHosts" : "prodServer",
                                  "compose_file": "${WORKSPACE}/docker-compose.yaml",
                                  "access_key": AWS_ACCESS_KEY_ID,
                                  "access_secret": AWS_SECRET_ACCESS_KEY
                              ]
                          )
                    }
                }
            }
    }
    post {
        always {
            archiveArtifacts artifacts: "trivy_report.html", fingerprint: true

            publishHTML (target: [
                allowMissing: false,
                alwaysLinkToLastBuild: false,
                keepAll: true,
                reportDir: '.',
                reportFiles: 'trivy_report.html',
                reportName: 'Trivy Scan',
                ])
            }
        }
}