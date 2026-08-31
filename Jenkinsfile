pipeline {

    agent any

    tools {
        maven 'maven'
    }

    environment {

        APP_NAME = 'java-web-app'
        WAR_FILE = 'target/java-web-app.war'

        S3_BUCKET = 'sam-java-cicd-artifacts-2026'
        S3_ARTIFACT = "artifacts/${APP_NAME}/${BUILD_NUMBER}/${APP_NAME}.war"

        TESTING_URL = 'http://172.31.40.160:8080'
        PRODUCTION_URL = 'http://172.31.42.86:8080'

        AWS_DEFAULT_REGION = 'ap-south-1'
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

        stage('Upload WAR to S3') {
            steps {

                sh '''
                    aws s3 cp \
                    "$WAR_FILE" \
                    "s3://$S3_BUCKET/$S3_ARTIFACT"
                '''
            }
        }

        stage('Deploy to Testing') {
            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'tomcat-testing',
                        usernameVariable: 'TOMCAT_USER',
                        passwordVariable: 'TOMCAT_PASSWORD'
                    )
                ]) {

                    sh '''
                        aws s3 cp \
                        "s3://$S3_BUCKET/$S3_ARTIFACT" \
                        "$WAR_FILE"

                        curl --fail \
                        --user "$TOMCAT_USER:$TOMCAT_PASSWORD" \
                        --upload-file "$WAR_FILE" \
                        "$TESTING_URL/manager/text/deploy?path=/$APP_NAME&update=true"
                    '''
                }
            }
        }

        stage('Production Approval') {
            steps {

                input(
                    message: 'Deploy the approved WAR artifact to Production?',
                    ok: 'Deploy to Production'
                )
            }
        }

        stage('Deploy to Production') {
            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'tomcat-production',
                        usernameVariable: 'TOMCAT_USER',
                        passwordVariable: 'TOMCAT_PASSWORD'
                    )
                ]) {

                    sh '''
                        aws s3 cp \
                        "s3://$S3_BUCKET/$S3_ARTIFACT" \
                        "$WAR_FILE"

                        curl --fail \
                        --user "$TOMCAT_USER:$TOMCAT_PASSWORD" \
                        --upload-file "$WAR_FILE" \
                        "$PRODUCTION_URL/manager/text/deploy?path=/$APP_NAME&update=true"
                    '''
                }
            }
        }
    }

    post {

        success {
            echo 'DevOps CI/CD pipeline completed successfully.'
        }

        failure {
            echo 'DevOps CI/CD pipeline failed.'
        }

        always {
            echo 'Pipeline execution completed.'
        }
    }
}
