pipeline {
    agent any
    tools {
        jdk 'jdk17'
    }

    environment {
        SONARQUBE_SCANNER_HOME = tool 'sonar-scanner'
        MAVEN_REPO_URL = 'https://mymavenrepo.com/repository/maven-releases/'
        MAVEN_REPO_USERNAME = credentials('mymaven-username')
        MAVEN_REPO_PASSWORD = credentials('mymaven-password')
        SLACK_WEBHOOK = credentials('slack-webhook-url')
    }

    stages {
        stage('Test') {
            steps {
                script {
                    sh './gradlew test'

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
        }

        stage('Code Analysis') {
            steps {
                script {
                    sh './gradlew sonar --info'
                }
            }
        }

        stage('Code Quality') {
            steps {
                script {
                    timeout(time: 10, unit: 'MINUTES') {
                        waitForQualityGate abortPipeline: true
                    }
                }
            }
        }

        stage('Build') {
            steps {
                script {
                    sh './gradlew build jar javadoc'

                    archiveArtifacts artifacts: 'build/libs/*.jar', fingerprint: true
                    archiveArtifacts artifacts: 'build/docs/javadoc/**', fingerprint: true
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    sh './gradlew publish'
                }
            }
        }
    }

    post {
        success {
            script {
                emailext (
                    subject: "[SUCCESS] Pipeline succeeded: ${env.JOB_NAME} [${env.BUILD_NUMBER}]",
                    body: "Check the build here: ${env.BUILD_URL}",
                    recipientProviders: [[$class: 'DevelopersRecipientProvider']]
                )
                slackSend channel: '#dev-notifications', message: ":white_check_mark: SUCCESS: Job ${env.JOB_NAME} #${env.BUILD_NUMBER} passed. See: ${env.BUILD_URL}"
            }
        }
        failure {
            script {

                emailext (
                    subject: "[FAILED] Pipeline failed: ${env.JOB_NAME} [${env.BUILD_NUMBER}]",
                    body: "Check the build here: ${env.BUILD_URL}",
                    recipientProviders: [[$class: 'DevelopersRecipientProvider']]
                )
                slackSend channel: '#dev-notifications', message: ":x: FAILED: Job ${env.JOB_NAME} #${env.BUILD_NUMBER} failed. See: ${env.BUILD_URL}"
            }
        }
    }
}