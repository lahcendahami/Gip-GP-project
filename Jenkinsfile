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
        stage('Set Version & GIB Reference') {
            steps {
                script {
                    def baseVersion = readMavenPom().getVersion().replaceAll('-SNAPSHOT$', '')
                    def gitHash = bat(script: '@git rev-parse --short=4 HEAD', returnStdout: true).trim()

                    switch (env.BRANCH_NAME) {
                        case 'main':
                        case 'master':
                            env.VERSION = "${baseVersion}-${gitHash}"
                            break
                        case ~/^release\/.*/:
                            env.VERSION = "${baseVersion}-${gitHash}-RELEASE"
                            break
                        case ~/^hotfix\/.*/:
                            env.VERSION = "${baseVersion}-${gitHash}-HOTFIX"
                            break
                        default:
                            def safeBranch = env.BRANCH_NAME.replaceAll('/', '-')
                            env.VERSION = "${baseVersion}-${gitHash}-${safeBranch}-SNAPSHOT"
                            break
                    }
                    echo "Docker image tag: ${env.VERSION}"

                    if (env.CHANGE_ID) {
                        env.GIB_ARGS = "-Dgib.referenceBranch=refs/remotes/origin/${env.CHANGE_TARGET}"

                    } else {
                        def lastSuccessful = env.GIT_PREVIOUS_SUCCESSFUL_COMMIT
                        if (lastSuccessful){
                            env.GIB_ARGS = "-Dgib.referenceBranch=${lastSuccessful}"
                        } else {
                            env.GIB_ARGS = "-Dgib.disable=true"
                        }
                    }
                    echo "GIB_ARGS: ${env.GIB_ARGS}"

                }
            }
        }

        stage('Build') {
            steps {
                bat "mvn clean install -DskipTests ${env.GIB_ARGS} -s ./.mvn/settings.xml"
            }
        }

        stage('Tests & Coverage') {
            steps {
                bat "mvn test ${env.GIB_ARGS} -s ./.mvn/settings.xml"
            }
        }
        stage('Publish Artifacts - Nexus') {
            when {
                anyOf {
                    branch 'develop'
                    branch 'main'
                    branch 'master'
                }
            }
            steps {
                bat "mvn deploy -Dmaven.test.skip=true ${env.GIB_ARGS} -s ./.mvn/settings.xml"
            }
        }

        stage('Build & Push Docker Images') {
            when {
                anyOf {
                    branch 'develop'
                    branch 'main'
                    branch 'master'
                    branch pattern: "release/*", comparator: "GLOB"
                    branch pattern: "hotfix/*", comparator: "GLOB"
                }
            }
            steps {
                bat "mvn jib:build -Dmaven.test.skip=true -Djib.to.tags=${env.VERSION} ${env.GIB_ARGS} -s ./.mvn/settings.xml"
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}
