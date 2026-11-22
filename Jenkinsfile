pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/DhiaAllagui/dhiaallagui.git'
                sh 'ls -la'
            }
        }
    }
}
