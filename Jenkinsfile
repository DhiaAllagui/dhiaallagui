pipeline {
    agent any

    // Names must match Manage Jenkins -> Tools
    tools {
        jdk 'JAVA_HOME'
        maven 'M2_HOME'
    }

    // Start a build automatically when GitHub sends a push webhook
    triggers {
        githubPush()
    }

    environment {
        IMAGE_NAME   = 'backend-app'
        IMAGE_TAG    = 'latest'
        NETWORK      = 'devops-net'
        BACKEND_PORT = '8082'   // 8080 = Jenkins, 8081 = docker-compose backend
    }

    stages {
        stage('Checkout') {
            steps {
                // 'github-creds' = ID of the credential in Manage Jenkins -> Credentials
                git branch: 'main',
                    url: 'https://github.com/DhiaAllagui/dhiaallagui.git',
                    credentialsId: 'github-creds'
            }
        }

        stage('Build & Test Maven') {
            steps {
                dir('backend') {
                    // Tests use an in-memory H2 database (src/test/resources)
                    sh 'mvn -B clean package'
                }
            }
        }

        stage('Docker Build') {
            steps {
                dir('backend') {
                    sh 'docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .'
                }
            }
        }

        stage('Docker Push') {
            steps {
                // 'dockerhub-creds' = Docker Hub username + access token
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                                                  usernameVariable: 'DOCKER_USER',
                                                  passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker tag ${IMAGE_NAME}:${IMAGE_TAG} $DOCKER_USER/${IMAGE_NAME}:${IMAGE_TAG}
                        docker push $DOCKER_USER/${IMAGE_NAME}:${IMAGE_TAG}
                    '''
                }
            }
        }

        stage('Deploy MySQL') {
            steps {
                sh '''
                    docker network create ${NETWORK} || true

                    if [ -n "$(docker ps -q -f name=^mysql$)" ]; then
                        echo "MySQL container already running"
                    else
                        docker rm -f mysql || true
                        docker run -d --name mysql --network ${NETWORK} --restart unless-stopped \
                            -e MYSQL_ROOT_PASSWORD=root \
                            -e MYSQL_DATABASE=test_db \
                            -v mysql-cd-data:/var/lib/mysql \
                            mysql:8.0
                    fi

                    echo "Waiting for MySQL..."
                    for i in $(seq 1 60); do
                        if docker exec mysql mysql -h127.0.0.1 -uroot -proot -e "SELECT 1" >/dev/null 2>&1; then
                            echo "MySQL is ready"; exit 0
                        fi
                        sleep 2
                    done
                    echo "MySQL did not start in time"; exit 1
                '''
            }
        }

        stage('Deploy backend-app') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                                                  usernameVariable: 'DOCKER_USER',
                                                  passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        # Stop and remove the old container if it exists
                        docker rm -f backend-app || true

                        # Get the published image from the registry
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker pull $DOCKER_USER/${IMAGE_NAME}:${IMAGE_TAG}

                        # Start a new container from that image
                        docker run -d --name backend-app --network ${NETWORK} --restart unless-stopped \
                            -p ${BACKEND_PORT}:8080 \
                            -e SPRING_DATASOURCE_URL="jdbc:mysql://mysql:3306/test_db?createDatabaseIfNotExist=true" \
                            -e SPRING_DATASOURCE_USERNAME=root \
                            -e SPRING_DATASOURCE_PASSWORD=root \
                            $DOCKER_USER/${IMAGE_NAME}:${IMAGE_TAG}
                    '''
                }
            }
        }

        stage('Verify') {
            steps {
                sh '''
                    docker ps

                    echo "Waiting for Spring Boot to start..."
                    for i in $(seq 1 30); do
                        if docker logs backend-app 2>&1 | grep -q "Started BackendApplication"; then
                            break
                        fi
                        sleep 3
                    done

                    docker logs backend-app

                    docker logs backend-app 2>&1 | grep -q "Started BackendApplication"
                    curl -sf http://localhost:${BACKEND_PORT}/entreprise/all && echo ""
                '''
            }
        }
    }

    post {
        always {
            sh 'docker logout || true'
        }
        success {
            echo 'CD pipeline finished: backend-app deployed.'
        }
    }
}
