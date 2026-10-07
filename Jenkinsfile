pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    environment {
        JAVA17_HOME = '/usr/lib/jvm/java-17-openjdk-amd64'

        IMAGE_NAME = 'devproject'
        CONTAINER_NAME = 'devproject'

        APP_PORT = '8081'
        CONTAINER_PORT = '8080'

        SONAR_PLUGIN = 'org.sonarsource.scanner.maven:sonar-maven-plugin:5.4.0.6343'
    }

    stages {

        stage('Environment Check') {
            steps {
                sh '''
                    set -e

                    echo "========================================"
                    echo "       ENVIRONMENT CHECK"
                    echo "========================================"

                    echo "===== Java 17 ====="
                    ${JAVA17_HOME}/bin/java -version

                    echo "===== Maven ====="
                    export JAVA_HOME=${JAVA17_HOME}
                    export PATH=${JAVA_HOME}/bin:/usr/bin:/bin
                    mvn -version

                    echo "===== Docker ====="
                    docker --version

                    echo "========================================"
                '''
            }
        }

        stage('Checkout') {
            steps {
                echo "===== Checking Out Source Code ====="
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh '''
                    set -e

                    export JAVA_HOME=${JAVA17_HOME}
                    export PATH=${JAVA_HOME}/bin:/usr/bin:/bin

                    echo "========================================"
                    echo "          MAVEN BUILD"
                    echo "========================================"

                    echo "JAVA_HOME=${JAVA_HOME}"

                    java -version
                    mvn -version

                    mvn clean package -DskipTests

                    echo "===== Maven Build Successful ====="
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                    set -e

                    export JAVA_HOME=${JAVA17_HOME}
                    export PATH=${JAVA_HOME}/bin:/usr/bin:/bin

                    echo "========================================"
                    echo "             TESTS"
                    echo "========================================"

                    mvn test

                    echo "===== Tests Passed ====="
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                        set -e

                        export JAVA_HOME=${JAVA17_HOME}
                        export PATH=${JAVA_HOME}/bin:/usr/bin:/bin

                        echo "========================================"
                        echo "       SONARQUBE ANALYSIS"
                        echo "========================================"

                        echo "SonarQube Server: ${SONAR_HOST_URL}"

                        java -version
                        mvn -version

                        mvn ${SONAR_PLUGIN}:sonar \
                            -Dsonar.projectKey=devproject \
                            -Dsonar.projectName=devproject \
                            -Dsonar.host.url=${SONAR_HOST_URL}

                        echo "===== SonarQube Analysis Completed ====="
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                echo "========================================"
                echo "       SONARQUBE QUALITY GATE"
                echo "========================================"

                timeout(time: 10, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }

                echo "===== Quality Gate Passed ====="
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    set -e

                    echo "========================================"
                    echo "          DOCKER BUILD"
                    echo "========================================"

                    docker build \
                        -t ${IMAGE_NAME}:${BUILD_NUMBER} \
                        -t ${IMAGE_NAME}:latest \
                        .

                    echo "===== Docker Image Built Successfully ====="

                    docker images ${IMAGE_NAME}
                '''
            }
        }

        stage('Docker Deploy') {
            steps {
                sh '''
                    set -e

                    echo "========================================"
                    echo "         DOCKER DEPLOY"
                    echo "========================================"

                    echo "===== Removing Old Container ====="

                    docker rm -f ${CONTAINER_NAME} 2>/dev/null || true

                    echo "===== Starting New Container ====="

                    docker run -d \
                        --name ${CONTAINER_NAME} \
                        --restart unless-stopped \
                        -p ${APP_PORT}:${CONTAINER_PORT} \
                        ${IMAGE_NAME}:${BUILD_NUMBER}

                    echo "===== Container Started ====="

                    sleep 10

                    echo "===== Container Status ====="

                    docker ps \
                        --filter "name=${CONTAINER_NAME}"

                    echo "===== Application Health Check ====="

                    if curl -f --max-time 10 http://localhost:${APP_PORT}; then
                        echo "===== Application is UP ====="
                    else
                        echo "===== Application Health Check FAILED ====="

                        echo "===== Docker Logs ====="

                        docker logs ${CONTAINER_NAME} --tail 100

                        exit 1
                    fi

                    echo "===== Deployment Successful ====="
                '''
            }
        }
    }

    post {

        success {
            echo '''
========================================
       PIPELINE SUCCESSFUL
========================================

Application:
http://65.2.56.162:8081

Jenkins:
http://65.2.56.162:8080

SonarQube:
http://65.2.56.162:9000

Docker Image:
devproject:${BUILD_NUMBER}

Docker Container:
devproject

========================================
'''
        }

        failure {
            echo '''
========================================
         PIPELINE FAILED
========================================
'''

            sh '''
                echo "===== Docker Containers ====="

                docker ps -a \
                    --filter "name=${CONTAINER_NAME}" || true

                echo "===== Docker Logs ====="

                docker logs ${CONTAINER_NAME} \
                    --tail 100 2>/dev/null || true
            '''
        }

        always {
            echo "Pipeline finished: ${currentBuild.currentResult}"
        }
    }
}
