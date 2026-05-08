pipeline {

    agent any

	tools {
        maven "MAVEN3"
        jdk "OracleJDK8"
    }

    environment {
        registry = "abdalrahman07/vprofileapp"
        registryCredential = "dockerhub"
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
}
