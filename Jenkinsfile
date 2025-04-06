pipeline {
environment { // Declaration of environment variables
DOCKER_ID = "laurencebel" // replace this with your docker-id
MOVIE_DOCKER_IMAGE = "movie-service"
CAST_DOCKER_IMAGE = "cast-service"
DOCKER_TAG = "v.${BUILD_ID}.0" // we will tag our images with the current build in order to increment the value by 1 with each new build
}
agent any // Jenkins will be able to select all available agents
stages {
        stage(' Docker Build'){ // docker build image stage
            steps {
                script {
                sh '''
                 docker rm -f jenkins
                 cd movie-service
                 docker build -t $DOCKER_ID/$MOVIE_DOCKER_IMAGE:$DOCKER_TAG .
                 cd ../cast-service
                 docker build -t $DOCKER_ID/$CAST_DOCKER_IMAGE:$DOCKER_TAG .
                sleep 6
                '''
                }
            }
        }
        stage('Docker run'){ // run container from our builded image
                steps {
                    script {
                    sh '''
                    sed -i "s+.*build: ./movie-service+    image: ${DOCKER_ID}/${MOVIE_DOCKER_IMAGE}:${DOCKER_TAG}+g" docker-compose.yml
                    sed -i "s+.*build: ./movie-service+    image: ${DOCKER_ID}/${CAST_DOCKER_IMAGE}:${DOCKER_TAG}+g" docker-compose.yml
                    docker compose up -d
                    sleep 10
                    '''
                    }
                }
            }

        stage('Test Acceptance'){ // we launch the curl command to validate that the container responds to the request
            steps {
                    script {
                    sh '''
                    curl http://localhost:9090/api/v1/movies/docs
                    curl http://localhost:9090/api/v1/casts/docs

                    curl -X 'POST' \
                        'http://34.240.190.249:30000/api/v1/casts/' \
                        -H 'accept: application/json' \
                        -H 'Content-Type: application/json' \
                        -d '{
                        "name": "tutu",
                        "nationality": "es"
                        }'

                    curl -X 'GET' \
                        'http://localhost:9090/api/v1/casts/1/' \
                        -H 'accept: application/json'

                    curl -X 'POST' \
                        'http://34.240.190.249:30000/api/v1/movies/' \
                        -H 'accept: application/json' \
                        -H 'Content-Type: application/json' \
                        -d '{
                        "name": "string",
                        "plot": "string",
                        "genres": [
                            "string"
                        ],
                        "casts_id": [
                            1
                        ]
                        }'
                    curl -X 'GET' \
                        'http://localhost:9090/api/v1/movies/' \
                        -H 'accept: application/json'

                    docker compose down
                    sleep 10
                    '''
                    }
            }

        }
        stage('Docker Push'){ //we pass the built image to our docker hub account
            environment
            {
                DOCKER_PASS = credentials("DOCKER_HUB_PASS") // we retrieve  docker password from secret text called docker_hub_pass saved on jenkins
            }

            steps {

                script {
                sh '''
                docker login -u $DOCKER_ID -p $DOCKER_PASS
                docker push $DOCKER_ID/$MOVIE_DOCKER_IMAGE:$DOCKER_TAG
                docker push $DOCKER_ID/$CAST_DOCKER_IMAGE:$DOCKER_TAG
                '''
                }
            }

        }

stage('Prepare Kube environment'){
        environment
        {
        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
        MOVIE_DB_PASSWORD = credentials("movie-db-password")
        MOVIE_DB_ID = credentials("movie-db-id")
        MOVIE_DB = "movie_db_dev"
        CAST_DB_PASSWORD = credentials("cast-db-password")
        CAST_DB_ID = credentials("cast-db-id")
        CAST_DB = "cast_db_dev"
        }
            steps {
                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                kubectl create configmap nginx-conf --from-file=./nginx_config.conf
                kubectl create secret generic movie-db-creds  --from-literal=POSTGRES_PASSWORD=$MOVIE_DB_PASSWORD --from-literal=POSTGRES_USER=$MOVIE_DB_ID --from-literal=POSTGRES_DB=$MOVIE_DB
                kubectl create secret generic cast-db-creds  --from-literal=POSTGRES_PASSWORD=$CAST_DB_PASSWORD --from-literal=POSTGRES_USER=$CAST_DB_ID --from-literal=POSTGRES_DB=$CAST_DB
                '''
                }
            }
        }


stage('Deploy') {
    parallel {

    stage('Deploiement en dev'){
            environment
            {
            NODE_PORT = "30000"
            NAMESPACE = "dev"
            }
                steps {
                    script {
                    sh '''
                    cd helm
                    helm upgrade --install movie-db postgres-movie/ --namespace $NAMESPACE
                    helm upgrade --install cast-db postgres-cast/ --namespace $NAMESPACE
                    cp api-service/values-movie.yaml values.yml
                    cat values.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                    helm upgrade --install movie-service movie-api-service --values=values.yml --namespace $NAMESPACE
                    cp api-service/values-cast.yaml values.yml
                    cat values.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                    helm upgrade --install cast-service cast-api-service --values=values.yml --namespace $NAMESPACE
                    helm upgrade --install nginx nginx/ --namespace dev --set service.type=NodePort --set service.nodePort=$NODE_PORT
                    '''
                    }
                }
            }

    stage('Deploiement en staging'){
            environment
            {
            NODE_PORT = "30001"
            NAMESPACE = "staging"
            }
                steps {
                    script {
                    sh '''
                    helm upgrade --install movie-db postgres-movie/ --namespace $NAMESPACE
                    helm upgrade --install cast-db postgres-cast/ --namespace $NAMESPACE
                    cp api-service/values-movie.yaml values.yml
                    cat values.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                    helm upgrade --install movie-service movie-api-service --values=values.yml --namespace $NAMESPACE
                    cp api-service/values-cast.yaml values.yml
                    cat values.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                    helm upgrade --install cast-service cast-api-service --values=values.yml --namespace $NAMESPACE
                    helm upgrade --install nginx nginx/ --namespace dev --set service.type=NodePort --set service.nodePort=$NODE_PORT
                    '''
                    }
                }

            }
    stage('Deploiement en qa'){
            environment
            {
            NODE_PORT = "30002"
            NAMESPACE = "qa"
            }
                steps {
                    script {
                    sh '''
                    helm upgrade --install movie-db postgres-movie/ --namespace $NAMESPACE
                    helm upgrade --install cast-db postgres-cast/ --namespace $NAMESPACE
                    cp api-service/values-movie.yaml values.yml
                    cat values.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                    helm upgrade --install movie-service movie-api-service --values=values.yml --namespace $NAMESPACE
                    cp api-service/values-cast.yaml values.yml
                    cat values.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                    helm upgrade --install cast-service cast-api-service --values=values.yml --namespace $NAMESPACE
                    helm upgrade --install nginx nginx/ --namespace dev --set service.type=NodePort --set service.nodePort=$NODE_PORT
                    '''
                    }
                }
            }
    }
    }

stage('Deploiement en prod'){
        when {
                branch 'master'
            }
        environment
        {
        NODE_PORT = "30003"
        NAMESPACE = "prod"
        }
            steps {
            // Create an Approval Button with a timeout of 15minutes.
            // this require a manuel validation in order to deploy on production environment
                    timeout(time: 15, unit: "MINUTES") {
                        input message: 'Do you want to deploy in production ?', ok: 'Yes'
                    }

                script {
                sh '''
                helm upgrade --install movie-db postgres-movie/ --namespace $NAMESPACE
                helm upgrade --install cast-db postgres-cast/ --namespace $NAMESPACE
                cp api-service/values-movie.yaml values.yml
                cat values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install movie-service movie-api-service --values=values.yml --namespace $NAMESPACE
                cp api-service/values-cast.yaml values.yml
                cat values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install cast-service cast-api-service --values=values.yml --namespace $NAMESPACE
                helm upgrade --install nginx nginx/ --namespace dev --set service.type=NodePort --set service.nodePort=$NODE_PORT
                '''
                }
            }

        }

        post { // send email when the job has failed
        // ..
        failure {
            echo "This will run if the job failed"
            mail to: "laurence.bellebouche@gmail.com",
                subject: "${env.JOB_NAME} - Build # ${env.BUILD_ID} has failed",
                body: "For more info on the pipeline failure, check out the console output at ${env.BUILD_URL}"
        }
        // ..
        always {
            script {
                sh '''
                kubectl delete secret cast-db-creds
                kubectl delete secret movie-db-creds
                kubectl delete configmap nginx-conf
                '''
        }
    }
}
}
