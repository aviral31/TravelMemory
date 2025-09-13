pipeline {
    agent { label 'Jenkins-Aviral' }
    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/aviral31/TravelMemory.git'
            }
        }
        stage('Install') {
            steps {
                sh 'cd backend; npm install'
            }
        }
        stage('Build') {
            steps {
                sh 'cd backend; npm run build'
            }
        }
    }

}





