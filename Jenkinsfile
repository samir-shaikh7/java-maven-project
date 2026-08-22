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
                            url: 'http://172.31.45.37:8080/'
                        )
                    ],
                    war: 'target/java-web-app.war'
                )
            }
        }
        
        stage('Smoke Test') {
            steps {
                sh '''
                    echo "Running staging smoke test..."
        
                    STATUS=$(curl -L -s -o /dev/null -w "%{http_code}" \
                    http://172.31.45.37:8080/java-web-app)
        
                    echo "Final HTTP Status: $STATUS"
        
                    if [ "$STATUS" -ne 200 ]; then
                        echo "Smoke Test FAILED"
                        exit 1
                    fi
        
                    echo "Smoke Test PASSED"
                '''
            }
        }

        stage('Production Approval') {
            steps {
                timeout(time: 10, unit: 'MINUTES') {
                    input message: 'Staging smoke test passed. Deploy to Production?',
                          ok: 'Approve Production Deployment'
                }
            }
        }
        
    }

    post {

        success {
            echo 'CI/CD pipeline completed successfully!'
        }

        failure {
            echo 'CI/CD pipeline failed!'
        }

        always {
            archiveArtifacts(
                artifacts: 'target/*.war',
                fingerprint: true,
                allowEmptyArchive: true
            )
        }
    }
}
