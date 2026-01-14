pipeline {
    agent any

    tools {
        jdk 'jdk11'
    }

    environment {
        MAVEN_REPO_URL = 'https://mymavenrepo.com/repository/maven-releases/'
        SONAR_HOST_URL = 'http://localhost:9000'
        PROJECT_NAME = 'TP7-OGL'
        PROJECT_VERSION = '1.0-SNAPSHOT'
    }

    stages {
        stage('Test') {
            steps {
                echo '========== Phase Test =========='
                bat 'gradlew clean test'

                junit 'build/test-results/test/TEST-*.xml'

                publishHTML([
                    allowMissing: false,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: 'build/reports/tests/test',
                    reportFiles: 'index.html',
                    reportName: 'Cucumber Test Report'
                ])
            }
        }

        stage('Code Analysis') {
            steps {
                echo '========== Phase Code Analysis =========='
                withSonarQubeEnv('SonarQube') {
                    bat 'gradlew sonar --info'
                }
            }
        }

        stage('Code Quality') {
            steps {
                echo '========== Phase Code Quality =========='
                timeout(time: 10, unit: 'MINUTES') {
                    script {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            error "Pipeline aborted: Quality Gate status = ${qg.status}"
                        }
                    }
                }
            }
        }

        stage('Build') {
            steps {
                echo '========== Phase Build =========='
                bat 'gradlew build jar javadoc'

                archiveArtifacts artifacts: 'build/libs/*.jar', fingerprint: true
                archiveArtifacts artifacts: 'build/docs/javadoc/**', fingerprint: true

                publishHTML([
                    allowMissing: false,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: 'build/docs/javadoc',
                    reportFiles: 'index.html',
                    reportName: 'Javadoc'
                ])
            }
        }

        stage('Deploy') {
            steps {
                echo '========== Phase Deploy =========='
                withCredentials([
                    usernamePassword(
                        credentialsId: 'mymaven-credentials',
                        usernameVariable: 'MAVEN_USERNAME',
                        passwordVariable: 'MAVEN_PASSWORD'
                    )
                ]) {
                    bat 'gradlew publish'
                }
                echo "Deployment successful to ${MAVEN_REPO_URL}"
            }
        }

        stage('Notification') {
            steps {
                echo '========== Phase Notification =========='

              emailext(
                  to: 'mb_bachferrag@esi.dz',
                  subject: "[SUCCESS] ${PROJECT_NAME} v${PROJECT_VERSION} deployed",
                  body: """
                      <html>
                      <body>
                          <h2 style="color:green;">Deployment Successful</h2>
                          <p><strong>Project:</strong> ${PROJECT_NAME}</p>
                          <p><strong>Version:</strong> ${PROJECT_VERSION}</p>
                          <p><strong>Build:</strong> #${env.BUILD_NUMBER}</p>
                          <p><a href="${env.BUILD_URL}">View Build Details</a></p>
                          <p><em>Sent by Jenkins CI/CD</em></p>
                      </body>
                      </html>
                  """,
                  mimeType: 'text/html'
              )

                // Notification Slack
                slackSend(
                    tokenCredentialId: 'slack-webhook-url',
                     channel: '#dev-notifications',
                       color: 'good',
                       message: "*SUCCESS*: Job ${env.JOB_NAME} #${env.BUILD_NUMBER} passed."

                )
            }
        }
    }

    post {
        failure {
            script {
                emailext (
                    subject: "[FAILED] ${PROJECT_NAME} #${env.BUILD_NUMBER}",
                    body: "Build failed. See: ${env.BUILD_URL}",
                    recipientProviders: [[$class: 'DevelopersRecipientProvider']]
                )
                slackSend(
                    channel: '#dev-notifications',
                    color: 'danger',
                    message: ":x: FAILED: ${PROJECT_NAME} #${env.BUILD_NUMBER}. ${env.BUILD_URL}"
                )
            }
        }
    }
}