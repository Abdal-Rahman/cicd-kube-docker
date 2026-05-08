pipeline {

    agent any

	tools {
        maven "MAVEN3"
        jdk "OracleJDK8"
    }

    environment {
        registry = "abdalrahman07/vprofileapp"
        registryCredential = "dockerhub"
        SLACK_CHANNEL = "#devops-notifications"
    }

    stages{

        stage('BUILD'){
            steps {
                sh 'mvn clean install -DskipTests'
            }
            post {
                success {
                    echo 'Now Archiving...'
                    archiveArtifacts artifacts: '**/target/*.war'
                }
            }
        }

        stage('UNIT TEST'){
            steps {
                sh 'mvn test'
            }
        }

        stage('INTEGRATION TEST'){
            steps {
                sh 'mvn verify -DskipUnitTests'
            }
        }

        stage ('CODE ANALYSIS WITH CHECKSTYLE'){
            steps {
                sh 'mvn checkstyle:checkstyle'
            }
            post {
                success {
                    echo 'Generated Analysis Result'
                }
            }
        }

        stage ('Build APP Image') {
            steps {
                script {
                    dockerImage = docker.build registry + ":V$BUILD_NUMBER"
                }
            }
        }

        stage ('TRIVY SECURITY SCAN') {
            steps {
                sh """
                    trivy image \
                        --exit-code 1 \
                        --severity HIGH,CRITICAL \
                        --no-progress \
                        ${registry}:V${BUILD_NUMBER}
                """
            }
            post {
                failure {
                    echo 'Trivy detected HIGH/CRITICAL vulnerabilities — pipeline aborted. Fix before pushing to registry.'
                }
            }
        }

        stage ('Upload Image') {
            steps {
                script {
                    docker.withRegistry('', registryCredential) {
                        dockerImage.push ("V$BUILD_NUMBER")
                        dockerImage.push ('latest')
                    }
                }
            }
        }

        stage ('Remove Unused Docker Image') {
            steps {
                sh "docker rmi $registry:V$BUILD_NUMBER"
            }
        }

        stage ('Kubernetes Deploy') {
            agent {label 'KOPS'}
            steps {
                sh "helm upgrade --install --force vprofile-stack helm/vprofilecharts --set appimage=${registry}:V${BUILD_NUMBER} --namespace prod"
            }
        }
    }

    post {
        success {
            slackSend(
                channel: env.SLACK_CHANNEL,
                color: 'good',
                message: """:white_check_mark: *BUILD SUCCESS*
*Job:* ${env.JOB_NAME} #${env.BUILD_NUMBER}
*Image:* ${registry}:V${env.BUILD_NUMBER} deployed to *prod*
*Duration:* ${currentBuild.durationString}
<${env.BUILD_URL}|View Build>"""
            )
        }
        failure {
            slackSend(
                channel: env.SLACK_CHANNEL,
                color: 'danger',
                message: """:x: *BUILD FAILED*
*Job:* ${env.JOB_NAME} #${env.BUILD_NUMBER}
*Failed Stage:* ${env.STAGE_NAME}
<${env.BUILD_URL}|View Build>"""
            )
        }
    }
}
