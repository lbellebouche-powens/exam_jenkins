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
                        'http://localhost:9090/api/v1/casts/' \
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
                        'http://localhost:9090/api/v1/movies/' \
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
                cat $KUBECONFIG > .kube/config
                kubectl create configmap nginx-conf --from-file=./nginx_config.conf --namespace dev
                kubectl create configmap nginx-conf --from-file=./nginx_config.conf --namespace staging
                kubectl create configmap nginx-conf --from-file=./nginx_config.conf --namespace qa
                kubectl create configmap nginx-conf --from-file=./nginx_config.conf --namespace prod
                kubectl create secret generic movie-db-creds  --from-literal=POSTGRES_PASSWORD=$MOVIE_DB_PASSWORD --from-literal=POSTGRES_USER=$MOVIE_DB_ID --from-literal=POSTGRES_DB=$MOVIE_DB --namespace dev
                kubectl create secret generic movie-db-creds  --from-literal=POSTGRES_PASSWORD=$MOVIE_DB_PASSWORD --from-literal=POSTGRES_USER=$MOVIE_DB_ID --from-literal=POSTGRES_DB=$MOVIE_DB --namespace staging
                kubectl create secret generic movie-db-creds  --from-literal=POSTGRES_PASSWORD=$MOVIE_DB_PASSWORD --from-literal=POSTGRES_USER=$MOVIE_DB_ID --from-literal=POSTGRES_DB=$MOVIE_DB --namespace qa
                kubectl create secret generic movie-db-creds  --from-literal=POSTGRES_PASSWORD=$MOVIE_DB_PASSWORD --from-literal=POSTGRES_USER=$MOVIE_DB_ID --from-literal=POSTGRES_DB=$MOVIE_DB --namespace prod
                kubectl create secret generic cast-db-creds  --from-literal=POSTGRES_PASSWORD=$CAST_DB_PASSWORD --from-literal=POSTGRES_USER=$CAST_DB_ID --from-literal=POSTGRES_DB=$CAST_DB --namespace dev
                kubectl create secret generic cast-db-creds  --from-literal=POSTGRES_PASSWORD=$CAST_DB_PASSWORD --from-literal=POSTGRES_USER=$CAST_DB_ID --from-literal=POSTGRES_DB=$CAST_DB --namespace staging
                kubectl create secret generic cast-db-creds  --from-literal=POSTGRES_PASSWORD=$CAST_DB_PASSWORD --from-literal=POSTGRES_USER=$CAST_DB_ID --from-literal=POSTGRES_DB=$CAST_DB --namespace qa
                kubectl create secret generic cast-db-creds  --from-literal=POSTGRES_PASSWORD=$CAST_DB_PASSWORD --from-literal=POSTGRES_USER=$CAST_DB_ID --from-literal=POSTGRES_DB=$CAST_DB --namespace prod
                '''
                }
            }
        }


    stage('Deploiement en dev'){
            environment
            {
            KUBECONFIG = credentials("config")
            NODE_PORT = "30000"
            NAMESPACE = "dev"
            }
                steps {
                    script {
                    sh '''
                    rm -Rf .kube
                    mkdir .kube
                    cat $KUBECONFIG > .kube/config
                    cd helm
                    helm upgrade --install movie-db postgres-movie/ --namespace $NAMESPACE
                    helm upgrade --install cast-db postgres-cast/ --namespace $NAMESPACE
                    cp movie-api-service/values.yaml values.yml
                    cat values.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                    helm upgrade --install movie-service movie-api-service --values=values.yml --namespace $NAMESPACE
                    cp cast-api-service/values.yaml values.yml
                    cat values.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                    helm upgrade --install cast-service cast-api-service --values=values.yml --namespace $NAMESPACE
                    helm upgrade --install nginx nginx/ --namespace $NAMESPACE --set service.type=NodePort --set service.nodePort=$NODE_PORT
                    cd ..
                    '''
                    }
                }
            }

    stage('Deploiement en staging'){
            environment
            {
            KUBECONFIG = credentials("config")
            NODE_PORT = "30001"
            NAMESPACE = "staging"
            }
                steps {
                    script {
                    sh '''
                    rm -Rf .kube
                    mkdir .kube
                    cat $KUBECONFIG > .kube/config
                    cd helm
                    helm upgrade --install movie-db postgres-movie/ --namespace $NAMESPACE
                    helm upgrade --install cast-db postgres-cast/ --namespace $NAMESPACE
                    cp movie-api-service/values.yaml values.yml
                    cat values.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                    helm upgrade --install movie-service movie-api-service --values=values.yml --namespace $NAMESPACE
                    cp cast-api-service/values.yaml values.yml
                    cat values.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                    helm upgrade --install cast-service cast-api-service --values=values.yml --namespace $NAMESPACE
                    helm upgrade --install nginx nginx/ --namespace $NAMESPACE --set service.type=NodePort --set service.nodePort=$NODE_PORT
                    cd ..
                    '''
                    }
                }

            }
    stage('Deploiement en qa'){
            environment
            {
            KUBECONFIG = credentials("config")
            NODE_PORT = "30002"
            NAMESPACE = "qa"
            }
                steps {
                    script {
                    sh '''
                    rm -Rf .kube
                    mkdir .kube
                    cat $KUBECONFIG > .kube/config
                    cd helm
                    helm upgrade --install movie-db postgres-movie/ --namespace $NAMESPACE
                    helm upgrade --install cast-db postgres-cast/ --namespace $NAMESPACE
                    cp movie-api-service/values.yaml values.yml
                    cat values.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                    helm upgrade --install movie-service movie-api-service --values=values.yml --namespace $NAMESPACE
                    cp cast-api-service/values.yaml values.yml
                    cat values.yml
                    sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                    helm upgrade --install cast-service cast-api-service --values=values.yml --namespace $NAMESPACE
                    helm upgrade --install nginx nginx/ --namespace $NAMESPACE --set service.type=NodePort --set service.nodePort=$NODE_PORT
                    cd ..
                    '''
                    }
                }
            }

stage('Deploiement en prod'){
        when {
                branch 'master'
            }
        environment
        {
        KUBECONFIG = credentials("config")
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
                rm -Rf .kube
                mkdir .kube
                cat $KUBECONFIG > .kube/config
                cd helm
                helm upgrade --install movie-db postgres-movie/ --namespace $NAMESPACE
                helm upgrade --install cast-db postgres-cast/ --namespace $NAMESPACE
                cp movie-api-service/values.yaml values.yml
                cat values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install movie-service movie-api-service --values=values.yml --namespace $NAMESPACE
                cp cast-api-service/values.yaml values.yml
                cat values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install cast-service cast-api-service --values=values.yml --namespace $NAMESPACE
                helm upgrade --install nginx nginx/ --namespace $NAMESPACE --set service.type=NodePort --set service.nodePort=$NODE_PORT
                cd ..
                '''
                }
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
                kubectl delete secret cast-db-creds --namespace dev
                kubectl delete secret cast-db-creds --namespace staging
                kubectl delete secret cast-db-creds --namespace qa
                kubectl delete secret cast-db-creds --namespace prod
                kubectl delete secret movie-db-creds --namespace dev
                kubectl delete secret movie-db-creds --namespace staging
                kubectl delete secret movie-db-creds --namespace qa
                kubectl delete secret movie-db-creds --namespace prod
                kubectl delete configmap nginx-conf --namespace dev
                kubectl delete configmap nginx-conf --namespace staging
                kubectl delete configmap nginx-conf --namespace qa
                kubectl delete configmap nginx-conf --namespace prod
                '''
            }
        }
    }
}
