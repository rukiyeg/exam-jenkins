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
 
                    echo "=== Etape 1 : demarrage des bases de donnees uniquement ==="
                    docker compose up -d movie_db cast_db
 
                    echo "=== Etape 2 : attente que Postgres soit pret (jusqu'a 60s) ==="
                    DB_READY=0
                    for i in $(seq 1 20); do
                        if docker compose exec -T movie_db pg_isready -q && docker compose exec -T cast_db pg_isready -q; then
                            DB_READY=1
                            echo "Bases de donnees pretes"
                            break
                        fi
                        echo "Postgres pas encore pret, tentative $i/20"
                        sleep 3
                    done
 
                    if [ "$DB_READY" -ne 1 ]; then
                        echo "ECHEC : les bases de donnees ne repondent pas apres 60 secondes"
                        docker compose logs movie_db cast_db
                        exit 1
                    fi
 
                    echo "=== Etape 3 : demarrage des services applicatifs et de nginx ==="
                    docker compose up --build -d movie_service cast_service nginx
 
                    echo "=== Etape 4 : attente des endpoints /docs (jusqu'a 90s, relance auto si besoin) ==="
                    READY=0
                    for i in $(seq 1 18); do
                        if curl -sf -o /dev/null http://localhost:8080/api/v1/movies/docs && \
                           curl -sf -o /dev/null http://localhost:8080/api/v1/casts/docs; then
                            READY=1
                            break
                        fi
                        echo "Pas encore pret, tentative $i/18 - relance des conteneurs applicatifs au cas ou ils auraient crashe"
                        docker compose up -d movie_service cast_service nginx
                        sleep 5
                    done
 
                    echo "=== Etat des conteneurs (DEBUG) ==="
                    docker compose ps -a
 
                    if [ "$READY" -ne 1 ]; then
                        echo "ECHEC : les endpoints /docs ne repondent pas apres 90 secondes"
                        docker compose logs
                        exit 1
                    fi
 
                    echo "=== movie-service /docs ==="
                    curl -i http://localhost:8080/api/v1/movies/docs
                    echo "=== cast-service /docs ==="
                    curl -i http://localhost:8080/api/v1/casts/docs
                '''
            }
        
            post {
                always {
                    sh 'docker compose down -v || true'
                }
            }
        }

    }
}