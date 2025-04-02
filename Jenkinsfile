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
                    cd ../
                    sed -i "s+.*build: ./movie-service+.*image: ${DOCKER_ID}/${MOVIE_DOCKER_IMAGE}:${DOCKER_TAG}+g" docker-compose.yml
                    sed -i "s+.*build: ./movie-service+.*image: ${DOCKER_ID}/${CAST_DOCKER_IMAGE}:${DOCKER_TAG}+g" docker-compose.yml
                    docker compose up
                    sleep 10
                    '''
                    }
                }
            }

        stage('Test Acceptance'){ // we launch the curl command to validate that the container responds to the request
            steps {
                    script {
                    sh '''
                    curl http://localhost:8080/api/v1/movies/docs
                    curl http://localhost:8080/api/v1/casts/docs
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

stage('Deploiement en dev'){
        environment
        {
        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
        }
            steps {
                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                cp movie-chart/values.yaml values.yml
                cat values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install movie-service movie-chart --values=values.yml --namespace dev
                cp cast-chart/values.yaml values.yml
                cat values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install cast-service cast-chart --values=values.yml --namespace dev
                '''
                }
            }

        }
stage('Deploiement en staging'){
        environment
        {
        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
        }
            steps {
                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                cp movie-chart/values.yaml values.yml
                cat values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install movie-service movie-chart --values=values.yml --namespace staging
                cp cast-chart/values.yaml values.yml
                cat values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install cast-service cast-chart --values=values.yml --namespace staging
                '''
                }
            }

        }
stage('Deploiement en qa'){
        environment
        {
        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
        }
            steps {
                script {
                sh '''
                rm -Rf .kube
                mkdir .kube
                ls
                cat $KUBECONFIG > .kube/config
                cp movie-chart/values.yaml values.yml
                cat values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install movie-service movie-chart --values=values.yml --namespace qa
                cp cast-chart/values.yaml values.yml
                cat values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install cast-service cast-chart --values=values.yml --namespace qa
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
        KUBECONFIG = credentials("config") // we retrieve  kubeconfig from secret file called config saved on jenkins
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
                ls
                cat $KUBECONFIG > .kube/config
                cp movie-chart/values.yaml values.yml
                cat values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install movie-service movie-chart --values=values.yml --namespace prod
                cp cast-chart/values.yaml values.yml
                cat values.yml
                sed -i "s+tag.*+tag: ${DOCKER_TAG}+g" values.yml
                helm upgrade --install cast-service cast-chart --values=values.yml --namespace prod
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
    }
}
