pipeline {

    agent any

    tools {
        maven 'maven'
    }

    environment {

        STAGING_URL = 'http://172.31.20.145:8080'
        PRODUCTION_URL = 'http://172.31.21.123:8080'

        APP_NAME = 'java-web-app'
        WAR_FILE = 'target/java-web-app.war'
    }

    stages {

        stage('Checkout') {
            steps {
                git(
                    branch: 'main',
                    url: 'https://github.com/samir-shaikh7/java-maven-project.git'
                )
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

                withSonarQubeEnv('SonarQube') {

                    withCredentials([
                        string(
                            credentialsId: 'sonarqube-token',
                            variable: 'SONAR_TOKEN'
                        )
                    ]) {

                        sh '''
                            mvn org.sonarsource.scanner.maven:sonar-maven-plugin:sonar \
                            -Dsonar.projectKey=java-web-app \
                            -Dsonar.projectName=java-web-app \
                            -Dsonar.token=$SONAR_TOKEN
                        '''
                    }
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

                sh 'ls -lh target/java-web-app.war'
            }
        }

        stage('Deploy to Testing-Server') {
            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'tomcat-testing',
                        usernameVariable: 'TOMCAT_USER',
                        passwordVariable: 'TOMCAT_PASSWORD'
                    )
                ]) {

                    sh '''
                        curl --fail \
                        --user "$TOMCAT_USER:$TOMCAT_PASSWORD" \
                        --upload-file "$WAR_FILE" \
                        "$STAGING_URL/manager/text/deploy?path=/$APP_NAME&update=true"
                    '''
                }
            }
        }

        stage('Smoke Test') {
            steps {

                sh '''
                    STATUS=$(curl -L -s -o /dev/null -w "%{http_code}" \
                    "$STAGING_URL/$APP_NAME")

                    echo "Staging HTTP Status: $STATUS"

                    if [ "$STATUS" != "200" ]; then
                        echo "Staging smoke test failed"
                        exit 1
                    fi

                    echo "Staging smoke test passed"
                '''
            }
        }

        stage('Production Approval') {
            steps {

                input(
                    message: 'Deploy to Production?',
                    ok: 'Deploy'
                )
            }
        }

        stage('Deploy to Production-Server') {
            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'tomcat-production',
                        usernameVariable: 'TOMCAT_USER',
                        passwordVariable: 'TOMCAT_PASSWORD'
                    )
                ]) {

                    sh '''
                        curl --fail \
                        --user "$TOMCAT_USER:$TOMCAT_PASSWORD" \
                        --upload-file "$WAR_FILE" \
                        "$PRODUCTION_URL/manager/text/deploy?path=/$APP_NAME&update=true"
                    '''
                }
            }
        }

        stage('Production Smoke Test') {
            steps {

                sh '''
                    STATUS=$(curl -L -s -o /dev/null -w "%{http_code}" \
                    "$PRODUCTION_URL/$APP_NAME")

                    echo "Production HTTP Status: $STATUS"

                    if [ "$STATUS" != "200" ]; then
                        echo "Production smoke test failed"
                        exit 1
                    fi

                    echo "Production smoke test passed"
                '''
            }
        }
    }

    post {

        success {
            echo 'CI/CD Pipeline completed successfully.'
        }

        failure {
            echo 'CI/CD Pipeline failed.'
        }

        always {
            echo 'Pipeline execution completed.'
        }
    }
}
