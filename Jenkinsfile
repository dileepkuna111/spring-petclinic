
pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 45, unit: 'MINUTES')
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
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
                    trivy --version
                '''
            }
        }

        stage('Build and Unit Test') {
            steps {
                sh './mvnw -B clean verify'
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
                        ./mvnw -B \
                        org.sonarsource.scanner.maven:sonar-maven-plugin:5.2.0.4988:sonar \
                        -Dsonar.projectKey=${SONAR_PROJECT_KEY}
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    catchError(
                        buildResult: 'SUCCESS',
                        stageResult: 'UNSTABLE',
                        catchInterruptions: true
                    ) {
                        timeout(time: 10, unit: 'MINUTES') {
                            def qg = waitForQualityGate(
                                abortPipeline: false
                            )

                            echo "Quality Gate Status: ${qg.status}"

                            if (qg.status != 'OK') {
                                error("Quality Gate did not pass: ${qg.status}")
                            }

                            echo 'Quality Gate PASSED'
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
                      --exit-code 0 \
                      --skip-dirs .git \
                      --skip-dirs target \
                      .
                '''
            }

            post {
                always {
                    archiveArtifacts(
                        artifacts: 'trivy-report.json',
                        allowEmptyArchive: true
                    )
                }
            }
        }

        stage('Archive Artifact') {
            steps {
                archiveArtifacts(
                    artifacts: 'target/*.jar',
                    fingerprint: true
                )
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully.'
            echo 'Review Quality Gate and Trivy reports.'
        }

        failure {
            echo 'Pipeline failed. Check console output.'
        }

        always {
            cleanWs()
        }
    }
}
