pipeline {
    agent any

    tools {
        jdk 'jdk17'
    }

    environment {
        // SonarQube scanner
        SONARQUBE_SCANNER_HOME = tool 'sonar-scanner'

        // Maven repo
        MAVEN_REPO_URL = 'https://mymavenrepo.com/repository/maven-releases/'

        // These will be injected as plain env vars in Jenkins job config
        MAVEN_REPO_USERNAME = 'myMavenRepo'
        MAVEN_REPO_PASSWORD = 'test0005'

        // For Slack: just use a plain webhook URL as env var
        SLACK_WEBHOOK_URL = 'https://hooks.slack.com/services/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'
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
                // Send email
                emailext (
                    subject: "[SUCCESS] Pipeline succeeded: ${env.JOB_NAME} [${env.BUILD_NUMBER}]",
                    body: "Check the build here: ${env.BUILD_URL}",
                    recipientProviders: [[$class: 'DevelopersRecipientProvider']]
                )
                // Send Slack message via curl
                sh '''
                    curl -X POST -H "Content-Type: application/json" \
                    -d "{\"text\":\":white_check_mark: SUCCESS: Job '${JOB_NAME}' #${BUILD_NUMBER} passed. See: ${BUILD_URL}\"}" \
                    "${SLACK_WEBHOOK_URL}"
                '''
            }
        }
        failure {
            script {
                emailext (
                    subject: "[FAILED] Pipeline failed: ${env.JOB_NAME} [${env.BUILD_NUMBER}]",
                    body: "Check the build here: ${env.BUILD_URL}",
                    recipientProviders: [[$class: 'DevelopersRecipientProvider']]
                )
                sh '''
                    curl -X POST -H "Content-Type: application/json" \
                    -d "{\"text\":\":x: FAILED: Job '${JOB_NAME}' #${BUILD_NUMBER} failed. See: ${BUILD_URL}\"}" \
                    "${SLACK_WEBHOOK_URL}"
                '''
            }
        }
    }
}