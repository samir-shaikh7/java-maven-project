pipeline {

    agent any

    options {
        skipDefaultCheckout(true)
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

        stage('Source Code Checkout') {
            steps {

                git(
                    branch: 'main',
                    url: 'https://github.com/samir-shaikh7/java-maven-project.git'
                )
            }
        }

        stage('Maven Build') {
            steps {

                sh 'mvn clean compile'
            }
        }

        stage('Code Quality Analysis') {
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

        stage('Quality Gate Validation') {
            steps {

                timeout(time: 5, unit: 'MINUTES') {

                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('WAR Artifact Packaging') {
            steps {

                sh 'mvn package -DskipTests'

                sh 'ls -lh "$WAR_FILE"'
            }
        }

        stage('Upload Artifact to S3') {
            steps {

                sh '''
                    aws s3 cp \
                    "$WAR_FILE" \
                    "s3://$S3_BUCKET/$S3_ARTIFACT"
                '''
            }
        }

        stage('Deploy to Testing Environment') {
            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'tomcat-testing',
                        usernameVariable: 'TOMCAT_USER',
                        passwordVariable: 'TOMCAT_PASSWORD'
                    )
                ]) {

                    sh '''
                        rm -f "$WAR_FILE"

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

        stage('Production Deployment Approval') {
            steps {

                input(
                    message: 'Deploy the approved artifact to Production?',
                    ok: 'Deploy to Production'
                )
            }
        }

        stage('Deploy to Production Environment') {
            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'tomcat-production',
                        usernameVariable: 'TOMCAT_USER',
                        passwordVariable: 'TOMCAT_PASSWORD'
                    )
                ]) {

                    sh '''
                        rm -f "$WAR_FILE"

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
}
