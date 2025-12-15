pipeline {
    agent any

    tools {
        jdk 'JAVA_HOME'
        maven 'M2_HOME'
    }

    environment {
        DOCKER_IMAGE       = "azizwhibi/devopspipeline:latest"
        DOCKER_REGISTRY    = "docker.io"
        DOCKER_CREDENTIALS = "1"                 // DockerHub credentials ID
    }

    stages {
        stage('Checkout Git') {
            steps {
                git branch: 'master', url: 'https://github.com/azizwhibi/devops.git'
            }
        }

        stage('Build with Maven') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('SonarQube Analysis') {
            environment {
                SCANNER_HOME = tool 'SonarScanner'   // Jenkins tool name
            }
            steps {
                withSonarQubeEnv('SonarQube') {      // Jenkins Sonar server name
                    sh '''
                      $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectKey=devops \
                        -Dsonar.sources=src \
                        -Dsonar.java.binaries=target/classes
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh "docker build -t ${DOCKER_IMAGE} -f docker/Dockerfile ."
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: DOCKER_CREDENTIALS,
                    passwordVariable: 'DOCKER_PASS',
                    usernameVariable: 'DOCKER_USER'
                )]) {
                    script {
                        int retries = 3
                        for (int i = 1; i <= retries; i++) {
                            try {
                                sh '''
                                    set -e
                                    export DOCKER_CLIENT_TIMEOUT=300
                                    export COMPOSE_HTTP_TIMEOUT=300
                                    echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                                    docker push ${DOCKER_IMAGE}
                                '''
                                echo "Docker push succeeded on attempt ${i}"
                                break
                            } catch (err) {
                                echo "Push attempt ${i} failed. Retrying..."
                                if (i == retries) {
                                    error("Docker push failed after ${retries} attempts")
                                }
                            }
                        }
                    }
                }
            }
        }
    }

    post {
        success { echo 'Pipeline completed successfully!' }
        failure { echo 'Pipeline failed. Check the logs!' }
    }
}
