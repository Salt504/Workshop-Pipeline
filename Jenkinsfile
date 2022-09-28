pipeline {
    agent {
        kubernetes {
            label 'pod-poderoso'
            containerTemplate {
                name 'contenedor-poderoso'
                image 'eclipse-temurin:17.0.3_7-jdk'
                ttyEnabled true
                args '--network ci --mount type=volume,source=ci-maven-home,target=/root/.m2'
            }
        }
    }

    environment {
        ORG_NAME = 'deors'
        APP_NAME = 'workshop-pipelines'
        APP_CONTEXT_ROOT = '/'
        APP_LISTENING_PORT = '8080'
        TEST_CONTAINER_NAME = "ci-${APP_NAME}-${BUILD_NUMBER}"
        DOCKER_HUB = credentials("${ORG_NAME}-docker-hub")
    }

    stages {
        stage('Compile') {
            steps {
                container('contenedor-poderoso') {
                    echo '-=- compiling project -=-'
                    sh './mvnw clean compile'
                }
            }
        }

        stage('Unit tests') {
            steps {
                container('contenedor-poderoso') {
                    echo '-=- execute unit tests -=-'
                    sh './mvnw test org.jacoco:jacoco-maven-plugin:report'
                    junit 'target/surefire-reports/*.xml'
                    jacoco execPattern: 'target/jacoco.exec'
                }
            }
        }

        stage('Mutation tests') {
            steps {
                container('contenedor-poderoso') {
                    echo '-=- execute mutation tests -=-'
                    sh './mvnw org.pitest:pitest-maven:mutationCoverage'
                }
            }
        }

        stage('Package') {
            steps {
                container('contenedor-poderoso') {
                    echo '-=- packaging project -=-'
                    sh './mvnw package -DskipTests'
                    archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                }
            }
        }

        stage('Build Docker image') {
            steps {
                    container('contenedor-poderoso') {
                    echo '-=- build Docker image -=-'
                    sh './mvnw docker:build'
                }
            }
        }

        stage('Run Docker image') {
            steps {
                container('contenedor-poderoso') {
                    echo '-=- run Docker image -=-'
                    sh "docker run --name ${TEST_CONTAINER_NAME} --detach --rm --network ci --expose ${APP_LISTENING_PORT} --expose 6300 --env JAVA_OPTS='-Dserver.port=${APP_LISTENING_PORT} -Dspring.profiles.active=ci -javaagent:/jacocoagent.jar=output=tcpserver,address=*,port=6300' ${ORG_NAME}/${APP_NAME}:latest"
                }
            }
        }

        stage('Integration tests') {
            steps {
                container('contenedor-poderoso') {
                    echo '-=- execute integration tests -=-'
                    sh "curl --retry 5 --retry-connrefused --connect-timeout 5 --max-time 5 http://${TEST_CONTAINER_NAME}:${APP_LISTENING_PORT}/${APP_CONTEXT_ROOT}/actuator/health"
                    sh "./mvnw failsafe:integration-test failsafe:verify -DargLine=\"-Dtest.target.server.url=http://${TEST_CONTAINER_NAME}:${APP_LISTENING_PORT}/${APP_CONTEXT_ROOT}\""
                    sh "java -jar target/dependency/jacococli.jar dump --address ${TEST_CONTAINER_NAME} --port 6300 --destfile target/jacoco-it.exec"
                    sh 'mkdir target/site/jacoco-it'
                    sh 'java -jar target/dependency/jacococli.jar report target/jacoco-it.exec --classfiles target/classes --xml target/site/jacoco-it/jacoco.xml'
                    junit 'target/failsafe-reports/*.xml'
                    jacoco execPattern: 'target/jacoco-it.exec'
                }
            }
        }

        stage('Performance tests') {
            steps {
                container('contenedor-poderoso') {
                    echo '-=- execute performance tests -=-'
                    sh "./mvnw jmeter:configure@configuration jmeter:jmeter jmeter:results -Djmeter.target.host=${TEST_CONTAINER_NAME} -Djmeter.target.port=${APP_LISTENING_PORT} -Djmeter.target.root=${APP_CONTEXT_ROOT}"
                    perfReport sourceDataFiles: 'target/jmeter/results/*.csv'
                }
            }
        }

        stage('Dependency vulnerability scan 1') {
            steps {
                container('contenedor-poderoso') {
                    echo '-=- run dependency vulnerability scan -=-'
                    sh './mvnw dependency-check:check'
                    //dependencyCheckPublisher
                }
            }
        }

        stage('Code inspection & quality gate') {
            steps {
                container('contenedor-poderoso') {
                    echo '-=- run code inspection & check quality gate -=-'
                    withSonarQubeEnv('ci-sonarqube') {
                        sh './mvnw sonar:sonar'
                    }
                 timeout(time: 10, unit: 'MINUTES') {
                        waitForQualityGate abortPipeline: true
                    }
                }
            }
        }

        stage('Dependency vulnerability scan 2') {
            steps {
                container('contenedor-poderoso') {
                    echo '-=- run dependency vulnerability scan -=-'
                    sh './mvnw dependency-check:check'
                    dependencyCheckPublisher failedTotalCritical: '4', unstableTotalCritical: '4', failedTotalHigh: '0', unstableTotalHigh: '0', failedTotalMedium: '5', unstableTotalMedium: '5'
                    script {
                        if (currentBuild.result == 'FAILURE') {
                         error('Dependency vulnerabilities exceed the configured threshold')
                        }
                    }
                }
            }
        }

        stage('Push Docker image') {
            steps {
                container('contenedor-poderoso') {
                    echo '-=- push Docker image -=-'
                    sh './mvnw docker:push'
                }
            }
        }
    }
}