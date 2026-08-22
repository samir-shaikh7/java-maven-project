pipeline {

    agent any

    tools {
        maven 'maven'
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
                        mvn org.sonarsource.scanner.maven:sonar-maven-plugin:5.7.0.6970:sonar \
                        -Dsonar.projectKey=java-web-app \
                        -Dsonar.projectName="Java Application - MyApp" \
                        -Dsonar.host.url="$SONAR_HOST_URL" \
                        -Dsonar.token="$SONAR_AUTH_TOKEN"
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

        stage('Deploy to Staging') {
            steps {
                deploy(
                    adapters: [
                        tomcat9(
                            credentialsId: 'tomcat-staging',
                            url: 'http://172.31.45.37:8080'
                        )
                    ],
                    contextPath: 'java-web-app',
                    war: 'target/java-web-app.war'
                )
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
