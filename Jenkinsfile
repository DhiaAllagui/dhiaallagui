pipeline {
    agent any

    // Names must match Manage Jenkins -> Tools
    tools {
        jdk 'jdk21'
        maven 'maven3'
    }

    // Start a build automatically when GitHub sends a push webhook
    triggers {
        githubPush()
    }

    stages {
        stage('Checkout') {
            steps {
                // 'github-creds' = ID of the credential in Manage Jenkins -> Credentials
                git branch: 'main',
                    url: 'https://github.com/DhiaAllagui/dhiaallagui.git',
                    credentialsId: 'github-creds'
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
