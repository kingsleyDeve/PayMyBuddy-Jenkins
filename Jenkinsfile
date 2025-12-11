pipeline {

    agent none

    environment {
        ID_DOCKER        = "${ID_DOCKER_PARAMS}"
        IMAGE_NAME       = "paymybuddy"
        IMAGE_TAG        = "latest"
        APP_NAME         = "kingsley"

        STG_API_ENDPOINT  = "ip10-0-57-8-d4bogugltosglhl3v92g-1993.direct.docker.labs.eazytraining.fr"
        STG_APP_ENDPOINT  = "ip10-0-57-8-d4bogugltosglhl3v92g-80.direct.docker.labs.eazytraining.fr"

        PROD_API_ENDPOINT = "ip10-0-57-9-d4bogugltosglhl3v92g-1993.direct.docker.labs.eazytraining.fr"
        PROD_APP_ENDPOINT = "ip10-0-57-9-d4bogugltosglhl3v92g-80.direct.docker.labs.eazytraining.fr"

        port = 80

        CONTAINER_IMAGE   = "${IMAGE_NAME}:${IMAGE_TAG}"
    }

    stages {

        

        stage('Unit Tests') {
            agent any
            steps {
                sh '''
                    chmod +x mvnw
                    ./mvnw clean install -DskipTests

                    ./mvnw -B test
                '''
            }
        }

       

        stage('Build & Run Docker Image') {
            agent any
            steps {
                sh """
                    docker stop ${IMAGE_NAME} && docker rm -f ${IMAGE_NAME}
                    docker stop mysql && docker rm -f mysql
                    docker network create paymybuddy-net || true
                    
                    echo 'Building Docker image'
                    docker build -t ${CONTAINER_IMAGE} .
                    pwd 
                    ls
                    docker run  --name mysql \
                     --network paymybuddy-net \
                -e MYSQL_ROOT_PASSWORD=password \
                -v ./var/jenkins_home/workspace/test/src/main/resources/database/create.sql:/docker-entrypoint-initdb.d/create.sql:ro \
                -p 3306:3306 \
                mysql:8.0
             
                    sleep 10
                    

                    docker ps 
    
                    echo 'Running container'
                    docker run --name ${IMAGE_NAME} \
                    --network paymybuddy-net \
                    -p 8081:8080 \
                    ${CONTAINER_IMAGE}

                    


                    echo 'Waiting for application startup'
                    sleep 5
                """
            }
        }

        stage('Test image') {
            agent any
            steps {
                sh """
                    echo 'Testing application on http://127.0.0.1:8081'
                    curl --retry 7 --retry-delay 3 http://127.0.0.1:8081
                """
            }
        }

        stage('Clean Container') {
            agent any
            steps {
                sh '''
                    docker stop paymybuddy || true
                    docker rm paymybuddy || true
                '''
            }
        }

        stage('Save Artefact') {
            agent any
            steps {
                sh '''
                    docker save ${CONTAINER_IMAGE} > /tmp/paymybuddy.tar
                '''
            }
        }

        stage('Login and Push Image on Docker Hub') {
            agent any
            environment {
                DOCKERHUB_CREDS = credentials('dockerhub-credentials')
            }
            steps {
                sh '''
                    echo "${DOCKERHUB_CREDS_PSW}" | docker login -u "${DOCKERHUB_CREDS_USR}" --password-stdin
                    docker push ${CONTAINER_IMAGE}
                '''
            }
        }

        stage('STAGING - Deploy app') {
            agent any
            steps {
                sh """
                    echo '{\"your_name\":\"${APP_NAME}\",\"container_image\":\"${CONTAINER_IMAGE}\",\"external_port\":\"${EXTERNAL_PORT}\",\"internal_port\":\"${INTERNAL_PORT}\"}' > data.json
                    curl -X POST http://${STG_API_ENDPOINT}/staging -H 'Content-Type: application/json' --data-binary @data.json
                """
            }
        }

        stage('PRODUCTION - Deploy app') {
            agent any
            when { branch 'main' }
            steps {
                sh """
                    curl -X POST http://${PROD_API_ENDPOINT}/prod -H 'Content-Type: application/json' \
                        -d '{\"your_name\":\"${APP_NAME}\",\"container_image\":\"${CONTAINER_IMAGE}\",\"external_port\":\"${EXTERNAL_PORT}\",\"internal_port\":\"${INTERNAL_PORT}\"}'
                """
            }
        }
    }

    post {
        always {
            junit 'target/surefire-reports/*.xml'
        }

        success {
            slackSend(
                color: '#00FF00',
                message: "SUCCESS: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' - PROD: http://${PROD_APP_ENDPOINT} - STAGING: http://${STG_APP_ENDPOINT}"
            )
        }

        failure {
            slackSend(
                color: '#FF0000',
                message: "FAILURE: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'"
            )
        }
    }
}
