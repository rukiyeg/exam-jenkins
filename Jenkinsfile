pipeline {
    agent any

    environment {
        DOCKERHUB_USER = "rgungoroglu"
        IMAGE_TAG = "${env.BUILD_ID}"
        KUBECONFIG = credentials('config')
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
        
        stage('Docker Push'){
            environment
            {
            DOCKER_PASS = credentials("DOCKER_HUB_PASS") 
            }
            steps {
                script {
                    sh '''
                    docker login -u $DOCKERHUB_USER -p $DOCKER_PASS
                    docker push ${DOCKERHUB_USER}/movie-service:${IMAGE_TAG}
                    docker push ${DOCKERHUB_USER}/cast-service:${IMAGE_TAG}
                    '''
                }
            }
        }
        
        stage('Deploy - dev') {
            steps { script { deployToEnv('dev') } }
        }
 
        stage('Deploy - QA') {
            steps { script { deployToEnv('qa') } }
        }
 
        stage('Deploy - staging') {
            steps { script { deployToEnv('staging') } }
        }
 
        stage('Deploy manuelle - Production') {
            when { 
                expression { env.GIT_BRANCH == 'origin/master' }
             }
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    input message: "Confirmez le déploiement en PRODUCTION ?", ok: "Yes"
                }
                script { deployToEnv('prod') }
            }
        }        
    } 
    post {
        always {
            sh 'docker logout || true'
        }
        success {
            echo "Pipeline terminé avec succès pour la branche ${env.GIT_BRANCH}"
            mail to: "r.gungoroglu@gmail.com",
                subject: "${env.JOB_NAME} - Build # ${env.BUILD_ID} has succeeded",
                body: "For more info on the pipeline success, check out the console output at ${env.BUILD_URL}"
        }
        failure {
            echo "Echec du pipeline sur la branche ${env.GIT_BRANCH}" 
            mail to: "r.gungoroglu@gmail.com",
                subject: "${env.JOB_NAME} - Build # ${env.BUILD_ID} has failed",
                body: "For more info on the pipeline failure, check out the console output at ${env.BUILD_URL}"
        
        }
    } 
}
 
// Fonction de déploiement Helm réutilisée pour chaque environnement
def deployToEnv(String namespace) {
   sh """
        kubectl create namespace ${namespace} --dry-run=client -o yaml | kubectl apply -f -

        # Postgres (movie-db, cast-db) : chart generique reutilise deux fois
        helm upgrade --install movie-db ./charts/postgres \
            --namespace ${namespace} \
            -f ./charts/postgres/values-movie-db.yaml \
            --wait --timeout 2m

        helm upgrade --install cast-db ./charts/postgres \
            --namespace ${namespace} \
            -f ./charts/postgres/values-cast-db.yaml \
            --wait --timeout 2m

        # Applications : chart generique reutilise deux fois (movie-service / cast-service)
        helm upgrade --install movie-service ./charts/fastapi \
            --namespace ${namespace} \
            --set image.repository=${DOCKERHUB_USER}/movie-service \
            --set image.tag=${IMAGE_TAG} \
            -f ./charts/fastapi/values-movie-service.yaml \
            --wait --timeout 3m

        helm upgrade --install cast-service ./charts/fastapi \
            --namespace ${namespace} \
            --set image.repository=${DOCKERHUB_USER}/cast-service \
            --set image.tag=${IMAGE_TAG} \
            -f ./charts/fastapi/values-cast-service.yaml \
            --wait --timeout 3m

        # nginx : chart dedie
        helm upgrade --install nginx ./charts/nginx \
            --namespace ${namespace} \
            --wait --timeout 2m
            
        kubectl get pods -n ${namespace}
        helm list -n ${namespace}

        NGINX_IP=\$(kubectl get svc nginx -n ${namespace} -o jsonpath='{.spec.clusterIP}')
        echo "nginx ClusterIP dans ${namespace} : \$NGINX_IP"
 
        READY=0
        for i in \$(seq 1 12); do
            if curl -sf -o /dev/null http://\${NGINX_IP}:8080/api/v1/movies/docs && \
               curl -sf -o /dev/null http://\${NGINX_IP}:8080/api/v1/casts/docs; then
                READY=1
                break
            fi
            echo "App pas encore joignable via nginx, tentative \$i/12"
            sleep 5
        done
 
        if [ "\$READY" -ne 1 ]; then
            echo "ECHEC : l'app ne repond pas via nginx dans ${namespace} apres deploiement"
            kubectl logs -l app=nginx -n ${namespace} --tail=50
            exit 1
        fi
        echo "Deploiement ${namespace} verifie OK : http://\${NGINX_IP}:8080/api/v1/movies/docs"
        echo "Deploiement ${namespace} verifie OK : http://\${NGINX_IP}:8080/api/v1/cast/docs"
    """
    
}