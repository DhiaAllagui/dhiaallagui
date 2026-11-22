pipeline {
    agent any

    // Names must match Manage Jenkins -> Tools
    tools {
        jdk 'jdk21'
        maven 'maven3'
    }

    environment {
        // Test database runs in Docker on port 3307 to avoid clashing with a local MySQL
        DB_URL = 'jdbc:mysql://localhost:3307/test_db?createDatabaseIfNotExist=true'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build Backend') {
            steps {
                dir('backend') {
                    sh 'mvn -B clean package -DskipTests'
                }
            }
        }

        stage('Test Backend') {
            steps {
                // Start a temporary MySQL for the Spring Boot tests
                sh '''
                    docker rm -f mysql-test || true
                    docker run -d --name mysql-test -e MYSQL_ROOT_PASSWORD=root -p 3307:3306 mysql:8.0
                    echo "Waiting for MySQL..."
                    for i in $(seq 1 60); do
                        if docker exec mysql-test mysql -h127.0.0.1 -uroot -proot -e "SELECT 1" >/dev/null 2>&1; then
                            echo "MySQL is ready"; break
                        fi
                        sleep 2
                    done
                '''
                dir('backend') {
                    sh 'mvn -B test -Dspring.datasource.url="$DB_URL"'
                }
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'backend/target/surefire-reports/*.xml'
                    sh 'docker rm -f mysql-test || true'
                }
            }
        }

        stage('Build Frontend') {
            steps {
                dir('frontend') {
                    sh 'npm ci'
                    sh 'npm run build'
                }
            }
        }

        stage('Archive') {
            steps {
                archiveArtifacts artifacts: 'backend/target/*.jar, frontend/dist/**', fingerprint: true
            }
        }
    }

    post {
        success {
            echo 'Pipeline finished successfully.'
        }
        failure {
            echo 'Pipeline failed. Check the stage logs above.'
        }
    }
}
