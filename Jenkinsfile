pipeline {
    agent any

    environment {
        DOCKERHUB_USER = "rgungoroglu"
        IMAGE_TAG = "${env.BUILD_ID}"
    }
    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }
   
        stage('Build images') {
            steps {
                sh '''
					docker rm -f jenkins
                    docker build -t ${DOCKERHUB_USER}/movie-service:${IMAGE_TAG} ./movie-service
                    docker build -t ${DOCKERHUB_USER}/cast-service:${IMAGE_TAG} ./cast-service
                '''
            }
        }

        stage('Test Acceptance') {
            steps {
                sh '''
					docker compose down -v || true
                    docker compose up --build -d
                    echo "=== movie-service /docs ==="
                    curl -i http://localhost:8080/api/v1/movies/docs
                    echo ""
                    echo "=== cast-service /docs ==="
                    curl -i http://localhost:8080/api/v1/casts/docs
                '''
            }
        }
        
        post {
            always {
                sh 'docker compose down -v || true'
            }
        }

    }
}