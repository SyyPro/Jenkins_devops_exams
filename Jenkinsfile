pipeline {
    environment {

        DOCKER_IMAGE_CAST = "cast-service"
        DOCKER_IMAGE_MOVIE = "movie-service"

        DOCKER_TAG_CAST = "cast-v.${BUILD_ID}.0"
        DOCKER_TAG_MOVIE = "movie-v.${BUILD_ID}.0"

        DOCKER_ID = "syypro"

    }

    agent any

    stages {
        stage('Docker Build of all images') {
            steps {
                script {
                    sh '''
                    if [ "$(docker ps -aq)" ]; then
                        docker rm -f $(docker ps -aq)
                    fi

                    # Build Image cast-service
                    # docker build -t $DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG_CAST ./cast-service || exit 1
                    docker build -t $DOCKER_ID/$DOCKER_IMAGE_CAST:latest ./cast-service || exit 1

                    # Build Image movie-service
                    # docker build -t $DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG_MOVIE ./movie-service || exit 1
                    docker build -t $DOCKER_ID/$DOCKER_IMAGE_MOVIE:latest ./movie-service || exit 1

                    sleep 5
                    '''
                }
            }
        }

        stage('Docker run') {
            steps {
                script {
                    sh '''
                    # cast-service on port 8083
                    # docker run -d -p 8083:8000 --name cast-service $DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG_CAST
                    docker run -d -p 8083:8000 --name cast-service $DOCKER_ID/$DOCKER_IMAGE_CAST:latest

                    # movie-service on port 8082
                    # docker run -d -p 8082:8000 --name movie-service $DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG_MOVIE
                    docker run -d -p 8082:8000 --name movie-service $DOCKER_ID/$DOCKER_IMAGE_MOVIE:latest

                    docker ps

                    sleep 5
                    '''
                }
            }
        }


        //         stage('Docker test') {
        //     steps {
        //         script {
        //             sh '''
        //             curl localhost:8083/api/v1/casts/docs
        //             curl localhost:8082/api/v1/movies/docs                    
        //             sleep 5
        //             '''
        //         }
        //     }
        // }

        stage('Docker Push') {
            environment {
                DOCKER_PASS = credentials("DOCKER_HUB_PASS") // Docker password from Jenkins credentials
            }

            steps {
                script {
                    sh '''
                    docker login -u $DOCKER_ID -p $DOCKER_PASS
                    # docker push $DOCKER_ID/$DOCKER_IMAGE_CAST:$DOCKER_TAG_CAST
                    docker push $DOCKER_ID/$DOCKER_IMAGE_CAST:latest
                    
                    # docker push $DOCKER_ID/$DOCKER_IMAGE_MOVIE:$DOCKER_TAG_MOVIE
                    docker push $DOCKER_ID/$DOCKER_IMAGE_MOVIE:latest
                    '''
                }
            }
        }

        stage('Configure Kubernetes Access') {
            environment {
                KUBECONFIG = credentials('config') // kubeconfig file from Jenkins credentials
            }
            steps {
                script {
                    sh '''
                    # delete kube directory
                    rm -Rf ~/.kube

                    # create kube directory and kubeconfig
                    mkdir -p ~/.kube
                    echo "$KUBECONFIG" > ~/.kube/config

                    # kubeconfig permissions
                    chmod 600 ~/.kube/config
                    '''
                }
            }
        }

        stage('Prepare Kubernetes Secrets') {
            steps {
                script {
                    sh '''
                    # delete secrets in namespaces
                    kubectl delete secret postgres-secret --namespace staging || echo "No Secret in staging"
                    kubectl delete secret postgres-secret --namespace dev || echo "No Secret in dev"
                    kubectl delete secret postgres-secret --namespace qa || echo "No Secret in qa"
                    kubectl delete secret postgres-secret --namespace prod || echo "No Secret in prod"
                    '''
                }
            }
        }

        stage('Deployment in dev') {
            environment {
                KUBECONFIG = credentials('config')
            }
            steps {
                script {
                    sh '''
                    ls -a

                    # configmap nginx dev
                    kubectl create configmap nginx-config --from-file=./nginx_config.conf -n dev --dry-run=client -o yaml | kubectl apply -f -

                    # delete kube directory
                    rm -Rf ~/.kube

                    # create kube directory and kubeconfig
                    mkdir -p ~/.kube
                    echo "$KUBECONFIG" > ~/.kube/config
                    chmod 600 ~/.kube/config

                    # Update YAML
                    cp cm-chart/values.yaml values.yml

                    # deploy helm dev
                    kubectl delete pvc postgres-cast-pvc -n dev || true
                    helm upgrade --install cm-api cm-chart --values=values.yml --namespace dev
                    '''
                }
            }
        }

        stage('Deployment in staging') {
            environment {
                KUBECONFIG = credentials('config')
            }
            steps {
                script {
                    sh '''
                    ls -a

                    # configmap nginx staging
                    kubectl create configmap nginx-config --from-file=./nginx_config.conf -n staging --dry-run=client -o yaml | kubectl apply -f -

                    # delete kube directory
                    rm -Rf ~/.kube

                    # create kube directory and kubeconfig
                    mkdir -p ~/.kube
                    echo "$KUBECONFIG" > ~/.kube/config
                    chmod 600 ~/.kube/config

                    # Update YAML
                    cp cm-chart/values.yaml values.yml

                    # deploy helm staging
                    kubectl delete pvc postgres-cast-pvc -n staging || true
                    helm upgrade --install cm-api cm-chart --values=values.yml --namespace staging
                    '''
                }
            }
        }

        stage('Deployment in prod') {
            environment {
                KUBECONFIG = credentials('config')
            }
            steps {
                timeout(time: 20, unit: "MINUTES") {
                    input message: 'Do you want to deploy in production?', ok: 'Yes'
                }

                script {
                    sh '''
                    ls -a

                    # configmap nginx prod
                    kubectl create configmap nginx-config --from-file=./nginx_config.conf -n prod --dry-run=client -o yaml | kubectl apply -f -

                    # delete kube directory
                    rm -Rf ~/.kube

                    # create kube directory and kubeconfig
                    mkdir -p ~/.kube
                    echo "$KUBECONFIG" > ~/.kube/config
                    chmod 600 ~/.kube/config

                    # Update YAML
                    cp cm-chart/values.yaml values.yml

                    # deploy helm prod
                    kubectl delete pvc postgres-cast-pvc -n prod || true
                    helm upgrade --install cm-api cm-chart --values=values.yml --namespace prod
                    '''
                }
            }
        }
    }
}