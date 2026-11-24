pipeline {
    agent any

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build images') {
            steps {
                sh 'docker build -t mern-server ./Server'
                sh 'docker build -t mern-client ./Client'
            }
        }

        stage('Deploy with Compose') {
            steps {
                sh 'docker compose down || true'
                sh 'docker compose up --build -d'
            }
        }

        stage('Verify') {
            steps {
                sh 'sleep 10'
                sh 'docker compose ps'
            }
        }
    }

    post {
        failure {
            sh 'docker compose logs || true'
        }
    }
}