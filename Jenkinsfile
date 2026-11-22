pipeline {
    agent any

    // Names must match Manage Jenkins -> Tools
    tools {
        jdk 'jdk21'
        maven 'maven3'
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/DhiaAllagui/dhiaallagui.git'
                sh 'ls -la'
            }
        }

        stage('Build Maven') {
            steps {
                dir('backend') {
                    sh 'mvn -B clean compile'
                }
            }
        }
    }
}
