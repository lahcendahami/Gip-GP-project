pipeline {
    agent any
    options {
        disableConcurrentBuilds()
    }
    environment {
        SONAR_TOKEN = credentials('sonar-token')
    }
    tools {
        maven 'maven'
        jdk 'jdk-17'
    }
    stages {
        stage('Determine Reference Branch') {
            steps {
                script {
                    if (env.CHANGE_ID) {
                        env.GIB_ARGS = "-Dgib.referenceBranch=refs/remotes/origin/${env.CHANGE_TARGET} -Dgib.fetchReferenceBranch=true"
                        echo "PR detected: comparing against origin/${env.CHANGE_TARGET}"
                    } else {
                        env.GIB_ARGS = "-Dgib.referenceBranch=HEAD~1"
                        echo "Push detected on ${env.BRANCH_NAME}: comparing against HEAD~1"
                    }
                }
            }
        }
        stage('Build') {
            steps {
                bat "mvn clean install -DskipTests ${env.GIB_ARGS}"
            }
        }
        stage('Test') {
            steps {
                bat "mvn test ${env.GIB_ARGS}"
            }
        }
        stage('Publish Artifacts - Nexus') {
            steps {
                bat "mvn deploy -Dmaven.test.skip=true ${env.GIB_ARGS}"
            }
        }
        stage('Build & Push Docker Images') {
            steps {
                bat "mvn jib:build -Dmaven.test.skip=true ${env.GIB_ARGS}"
            }
        }
    }
}