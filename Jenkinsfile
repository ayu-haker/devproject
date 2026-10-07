pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 40, unit: 'MINUTES')
        buildDiscarder(logRotator(numToKeepStr: '10'))
        disableConcurrentBuilds()
        skipDefaultCheckout(true)
    }

    tools {
        jdk 'jdk17'
        maven 'maven3'
    }

    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKER_USER  = 'ayu-haker'
        APP_NAME     = 'devproject'
        K8S_SERVER   = 'https://172.31.40.204:6443'
        K8S_NS       = 'webapps'
    }

    stages {

        stage('Git Checkout') {
            steps {
                git(
                    branch: 'main',
                    credentialsId: 'git-cred',
                    url: 'https://github.com/ayu-haker/devproject.git'
                )
            }
        }

        stage('Compile') {
            steps {
                sh 'mvn -B clean compile'
            }
        }

        stage('Test') {
            steps {
                sh 'mvn -B test'
            }
            post {
                always {
                    junit testResults: '**/target/surefire-reports/*.xml', allowEmptyResults: true
                }
            }
        }

        stage('File System Scan') {
            steps {
                sh '''
                    trivy fs \
                    --format table \
                    -o trivy-fs-report.html \
                    .
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-fs-report.html', allowEmptyArchive: true
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar') {
                    sh '''
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=DevOpsProject \
                        -Dsonar.projectKey=DevOpsProject \
                        -Dsonar.java.binaries=.
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    timeout(unit: 'MINUTES', time: 10) {
                        waitForQualityGate(
                            abortPipeline: true,
                            credentialsId: 'sonar-token'
                        )
                    }
                }
            }
        }

        stage('Build') {
            steps {
                sh 'mvn -B package -DskipTests'
            }
        }

        stage('Build & Tag Docker Image') {
            steps {
                script {
                    withDockerRegistry(
                        credentialsId: 'docker-cred'
                    ) {
                        sh """
                            docker build -t ${APP_NAME}:${BUILD_ID} .
                            docker tag ${APP_NAME}:${BUILD_ID} ${DOCKER_USER}/${APP_NAME}:${BUILD_ID}
                            docker tag ${APP_NAME}:${BUILD_ID} ${DOCKER_USER}/${APP_NAME}:latest
                        """
                    }
                }
            }
        }

        stage('Docker Image Scan') {
            steps {
                sh """
                    trivy image \
                    --format table \
                    -o trivy-image-report.html \
                    ${DOCKER_USER}/${APP_NAME}:${BUILD_ID}
                """
            }
            post {
                always {
                    archiveArtifacts artifacts: 'trivy-image-report.html', allowEmptyArchive: true
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    withDockerRegistry(
                        credentialsId: 'docker-cred'
                    ) {
                        sh """
                            docker push ${DOCKER_USER}/${APP_NAME}:${BUILD_ID}
                            docker push ${DOCKER_USER}/${APP_NAME}:latest
                        """
                    }
                }
            }
        }

        stage('Deploy To Kubernetes') {
            steps {
                withKubeConfig(
                    credentialsId: 'k8-cred',
                    namespace: env.K8S_NS,
                    serverUrl: env.K8S_SERVER
                ) {
                    sh """
                        kubectl apply -f deployment-service.yaml
                        kubectl rollout status deployment/boardgame-deployment -n ${K8S_NS} --timeout=180s
                    """
                }
            }
        }
    }

    post {
        success {
            echo "Pipeline succeeded -> ${DOCKER_USER}/${APP_NAME}:${BUILD_ID}"
        }
        failure {
            echo "Pipeline failed for build #${BUILD_NUMBER}"
        }
        always {
            emailext(
                subject: "Jenkins Build #${BUILD_NUMBER} - ${JOB_NAME} [${currentBuild.currentResult}]",
                body: """
                    <p>Build Status: ${currentBuild.currentResult}</p>
                    <p>
                        Check the build details:
                        <a href="${BUILD_URL}">${BUILD_URL}</a>
                    </p>
                """,
                to: 'sunielmahla@gmail.com',
                replyTo: 'sunielmahla@gmail.com',
                mimeType: 'text/html'
            )
            cleanWs()
        }
    }
}
