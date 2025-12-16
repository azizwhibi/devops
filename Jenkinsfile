pipeline {
    agent any

    tools {
        jdk 'JAVA_HOME'
        maven 'M2_HOME'
    }

    environment {
        DOCKER_REGISTRY    = "192.168.33.10:8081"
        DOCKER_REPO        = "docker-hosted"
        DOCKER_IMAGE       = "${DOCKER_REGISTRY}/${DOCKER_REPO}/devopspipeline:latest"
        DOCKER_CREDENTIALS = "nexus-creds"
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
                SCANNER_HOME = tool 'SonarScanner'
            }
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                      echo "SCANNER_HOME=$SCANNER_HOME"
                      ls -R "$SCANNER_HOME"

                      "$SCANNER_HOME/bin/sonar-scanner" \
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
                    passwordVariable: 'NEXUS_PASS',
                    usernameVariable: 'NEXUS_USER'
                )]) {
                    script {
                        int retries = 3
                        for (int i = 1; i <= retries; i++) {
                            try {
                                sh """
                                    set -e
                                    export DOCKER_CLIENT_TIMEOUT=300
                                    export COMPOSE_HTTP_TIMEOUT=300
                                    echo "\$NEXUS_PASS" | docker login ${DOCKER_REGISTRY} -u "\$NEXUS_USER" --password-stdin
                                    docker push ${DOCKER_IMAGE}
                                    docker logout ${DOCKER_REGISTRY}
                                """
                                echo "Docker push to Nexus succeeded on attempt ${i}"
                                break
                            } catch (err) {
                                echo "Push attempt ${i} to Nexus failed. Retrying..."
                                if (i == retries) {
                                    error("Docker push to Nexus failed after ${retries} attempts")
                                }
                            }
                        }
                    }
                }
            }
        }
stage('Deploy to Kubernetes') {
    steps {
        sh '''
          export KUBECONFIG=/var/lib/jenkins/.kube/config

          kubectl get nodes
          kubectl apply -n devops -f /home/vagrant/kub/mysql-deployment.yaml
          kubectl apply -n devops -f /home/vagrant/kub/spring-deployment.yaml
        '''
    }
}


    }

    post {
        success { echo 'Pipeline completed successfully!' }
        failure { echo 'Pipeline failed. Check the logs!' }
    }
}
