pipeline {

    agent any

    tools {
        maven 'Maven-3.9.11'
    }

    environment {
        SONARQUBE = 'SonarQube'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean compile'
            }
        }

        stage('Unit Test') {
            steps {
                sh 'mvn test'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE}") {
                    sh '''
                        mvn sonar:sonar \
                        -Dsonar.projectKey=java-web-app \
                        -Dsonar.projectName="Java Application - Java-Web-App"
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Package WAR') {
            steps {
                sh 'mvn package -DskipTests'
            }
        }

    }

    post {

        success {
            echo 'CI Pipeline completed successfully!'
        }

        failure {
            echo 'CI Pipeline failed!'
        }

        always {
            archiveArtifacts artifacts: 'target/*.war',
                             fingerprint: true,
                             allowEmptyArchive: true
        }
    }
}
