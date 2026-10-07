pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    environment {
        IMAGE_NAME = 'devproject'
        CONTAINER_NAME = 'devproject'
        APP_PORT = '8081'
        CONTAINER_PORT = '8080'
    }

    stages {

        stage('Environment Check') {
            steps {
                sh '''
                    set -e

                    echo "===== Java Version ====="
                    java -version

                    echo "===== Maven Version ====="
                    mvn -version

                    echo "===== Docker Version ====="
                    docker --version
                '''
            }
        }

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh '''
                    set -e

                    echo "===== Maven Build ====="
                    mvn clean package -DskipTests

                    echo "===== Build Successful ====="
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                    set -e

                    echo "===== Running Tests ====="
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

                        echo "===== SonarQube Analysis ====="

                        mvn sonar:sonar \
                          -Dsonar.projectKey=devproject \
                          -Dsonar.projectName=devproject

                        echo "===== SonarQube Analysis Completed ====="
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                echo "===== Waiting for SonarQube Quality Gate ====="

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

                    echo "===== Building Docker Image ====="

                    docker build \
                      -t ${IMAGE_NAME}:${BUILD_NUMBER} \
                      -t ${IMAGE_NAME}:latest \
                      .

                    echo "===== Docker Build Successful ====="
                '''
            }
        }

        stage('Docker Deploy') {
            steps {
                sh '''
                    set -e

                    echo "===== Stopping Existing Container ====="

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
                    docker ps --filter "name=${CONTAINER_NAME}"

                    echo "===== Application URL ====="
                    echo "http://65.2.56.162:${APP_PORT}"
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
                echo "===== Docker Status ====="
                docker ps -a --filter "name=${CONTAINER_NAME}" || true

                echo "===== Docker Logs ====="
                docker logs ${CONTAINER_NAME} --tail 100 2>/dev/null || true
            '''
        }

        always {
            echo "Pipeline finished: ${currentBuild.currentResult}"
        }
    }
}
