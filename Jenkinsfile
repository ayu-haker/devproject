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

        // Jenkins = 8080
        // Application = 8081
        APP_PORT = '8081'
        CONTAINER_PORT = '8080'
    }

    stages {

        stage('Environment Check') {
            steps {
                sh '''
                    set -e

                    export JAVA_HOME=${JAVA17_HOME}
                    export PATH=${JAVA_HOME}/bin:${PATH}

                    echo "===== Java ====="
                    java -version

                    echo "===== Maven ====="
                    mvn -version

                    echo "===== Docker ====="
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

                    export JAVA_HOME=${JAVA17_HOME}
                    export PATH=${JAVA_HOME}/bin:${PATH}

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

                    export JAVA_HOME=${JAVA17_HOME}
                    export PATH=${JAVA_HOME}/bin:${PATH}

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

                        export JAVA_HOME=${JAVA17_HOME}
                        export PATH=${JAVA_HOME}/bin:${PATH}

                        echo "===== SonarQube Analysis ====="
                        echo "SonarQube: ${SONAR_HOST_URL}"

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

                    echo "===== Docker Build ====="

                    docker build \
                        -t ${IMAGE_NAME}:${BUILD_NUMBER} \
                        -t ${IMAGE_NAME}:latest \
                        .

                    echo "===== Docker Image Created ====="

                    docker images ${IMAGE_NAME}
                '''
            }
        }

        stage('Docker Deploy') {
            steps {
                sh '''
                    set -e

                    echo "===== Deploying Container ====="

                    docker rm -f ${CONTAINER_NAME} 2>/dev/null || true

                    docker run -d \
                        --name ${CONTAINER_NAME} \
                        --restart unless-stopped \
                        -p ${APP_PORT}:${CONTAINER_PORT} \
                        ${IMAGE_NAME}:${BUILD_NUMBER}

                    echo "===== Waiting for Application ====="

                    sleep 10

                    echo "===== Container Status ====="

                    docker ps --filter "name=${CONTAINER_NAME}"

                    echo "===== Application Health ====="

                    curl -f http://localhost:${APP_PORT} || {
                        echo "Application health check failed"
                        docker logs ${CONTAINER_NAME} --tail 100
                        exit 1
                    }

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

Docker:
devproject:${BUILD_NUMBER}

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
                docker ps -a --filter "name=${CONTAINER_NAME}" || true

                echo "===== Application Logs ====="
                docker logs ${CONTAINER_NAME} --tail 100 2>/dev/null || true
            '''
        }

        always {
            echo "Pipeline finished: ${currentBuild.currentResult}"
        }
    }
}
