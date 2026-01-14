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

             script {
                 mail(
                     to: 'mb_bachferrag@esi.dz',
                     subject: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                     body: """
             Déploiement réussi !
             Projet : ${env.JOB_NAME}
             Build : #${env.BUILD_NUMBER}
             Status : SUCCESS
             Date : ${new Date().format('dd/MM/yyyy HH:mm')}
             Voir le build : ${env.BUILD_URL}
                     """.stripIndent()
                 )
             }

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