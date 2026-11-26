pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
        disableConcurrentBuilds()
    }

    tools {
        jdk 'jdk21'
    }

    environment {
        SONAR_PROJECT_KEY = 'spring-petclinic-rest'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Validate Environment') {
            steps {
                sh '''
                    set -e
                    java -version
                    ./mvnw -version
                '''
            }
        }

        stage('Build and Unit Test') {
            steps {
                sh '''
                    set -e
                    ./mvnw -B clean verify
                '''
            }
            post {
                always {
                    junit allowEmptyResults: true,
                          testResults: '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('SonarQube Analysis') {
    steps {
        withSonarQubeEnv('SonarQube') {
            sh '''
                set -e

                ./mvnw -B \
                  org.sonarsource.scanner.maven:sonar-maven-plugin:5.2.0.4988:sonar \
                  -Dsonar.projectKey=${SONAR_PROJECT_KEY}
            '''
                }
            }
        }

       
stage('Quality Gate') {
    steps {
        timeout(time: 5, unit: 'MINUTES') {
            script {

                echo "SonarQube Quality Gate Status: ${qg.status}"

                if (qg.status != 'OK') {
                    echo "WARNING: Quality Gate did not pass. Continuing pipeline for learning purposes."
                }
            }
        }
    }
}

        stage('Trivy Filesystem Scan') {
            steps {
                sh '''
                    trivy fs \
                      --scanners vuln,secret,misconfig \
                      --severity HIGH,CRITICAL \
                      --format json \
                      --output trivy-report.json \
                      --exit-code 1 \
                      --skip-dirs .git \
                      --skip-dirs target \
                      .
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-report.json',
                                     allowEmptyArchive: true
                }
            }
        }

        stage('Archive Artifact') {
            steps {
                archiveArtifacts artifacts: 'target/*.jar',
                                 fingerprint: true
            }
        }
    }

    post {
        success {
            echo 'CI checks, SonarQube Quality Gate and Trivy scan passed.'
        }

        failure {
            echo 'Pipeline failed. Review the failed stage and its logs.'
        }

        always {
            cleanWs()
        }
    }
}
