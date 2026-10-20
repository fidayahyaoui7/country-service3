pipeline {
    agent any

    tools {
        jdk 'JDK21'
        maven 'M2_HOME'
    }

    environment {
        // Mettre 'true' quand Nexus est installé et configuré
        NEXUS_READY = 'true'
        NEXUS_URL   = 'http://localhost:8081'
    }

    stages {

        stage('Checkout code') {
            steps {
                git branch: 'main', url: 'https://github.com/fidayahyaoui7/country-service3.git'
            }
        }

        stage('Compile, test, package') {
            steps {
                sh 'mvn clean install'
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('MySonarQubeServer') {
                    // Nom complet du plugin (évite "No plugin found for prefix 'sonar'")
                    // "verify" regénère aussi le rapport JaCoCo lu par Sonar
                    sh 'mvn clean verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=countryservice3 -Dsonar.projectName=countryservice3'
                }
            }
        }

        stage('display message') {
            steps {
                echo 'Hello from Git'
            }
        }

        stage('Deploy jar to Nexus') {
            when { expression { env.NEXUS_READY == 'true' } }
            steps {
                withCredentials([usernamePassword(credentialsId: 'nexus-credentials', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                    sh 'mvn deploy -DskipTests -s settings.xml'
                }
            }
        }

        stage('Store reports in Nexus') {
            when { expression { env.NEXUS_READY == 'true' } }
            steps {
                sh '''
                    zip -r jacoco-report.zip target/site/jacoco
                    zip -r surefire-reports.zip target/surefire-reports
                '''
                withCredentials([usernamePassword(credentialsId: 'nexus-creds', usernameVariable: 'NEXUS_USER', passwordVariable: 'NEXUS_PASS')]) {
                    sh '''
                        curl --fail -u "$NEXUS_USER:$NEXUS_PASS" --upload-file jacoco-report.zip \
                          "$NEXUS_URL/repository/reports/countryservice3/$BUILD_NUMBER/jacoco-report.zip"
                        curl --fail -u "$NEXUS_USER:$NEXUS_PASS" --upload-file surefire-reports.zip \
                          "$NEXUS_URL/repository/reports/countryservice3/$BUILD_NUMBER/surefire-reports.zip"
                    '''
                }
            }
        }
    }
}