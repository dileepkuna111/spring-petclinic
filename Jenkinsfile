
pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    tools {
        jdk 'jdk21'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git log -1 --oneline'
            }
        }

        stage('Validate Environment') {
            steps {
                sh '''
                    java -version
                    git --version
                    test -f pom.xml
                    test -f mvnw
                    chmod +x mvnw
                '''
            }
        }

        stage('Build and Unit Test') {
            steps {
                sh './mvnw -B clean verify'
            }
        }

        stage('Verify Artifact') {
            steps {
                sh '''
                    ls -lh target/
                    find target -maxdepth 1 \
                      -name "*.jar" -type f
                '''
            }
        }
    }

    post {
        success {
            echo 'CI pipeline completed successfully!'
        }

        failure {
            echo 'CI pipeline failed. Check console logs.'
        }

        always {
            junit allowEmptyResults: true,
                testResults: 'target/surefire-reports/*.xml'

            archiveArtifacts \
                artifacts: 'target/*.jar',
                allowEmptyArchive: true
        }
    }
}
