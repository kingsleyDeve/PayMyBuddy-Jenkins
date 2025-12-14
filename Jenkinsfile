pipeline {

    agent none

    environment {
        ID_DOCKER        = "${ID_DOCKER_PARAMS}"
        IMAGE_NAME       = "kingsley95/paymybuddy"
        IMAGE_TAG        = "latest"
        APP_NAME         = "kingsley"
        IMAGE_MYSQL      = "kingsley95/paymybuddy-db"
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
                    head -5 /var/jenkins_home/workspace/test/src/main/resources/database/create.sql

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
            # Nettoyage préalable
            docker stop ${IMAGE_NAME} || true
            docker rm -f ${IMAGE_NAME} || true
            docker stop mysql || true
            docker rm  mysql || true
            docker network create paymybuddy-network || true
            docker build -t ${CONTAINER_IMAGE} .
            docker build -f Dockerfile-db -t ${IMAGE_MYSQL} .

    docker run --name mysql --network paymybuddy-net \
    -e MYSQL_ROOT_PASSWORD=pass \
    -e MYSQL_PASSWORD=pass \
    -e MYSQL_USER=tes \
    -e MYSQL_DATABASE=db_paymybuddy \
    -p 3306:3306 -d ${IMAGE_MYSQL}        

    sleep 30


            docker run --name ${IMAGE_NAME} \
                --network paymybuddy-net \
                -p 8081:8080 \
                --link mysql \
                -d \
                ${CONTAINER_IMAGE}
            sleep 10
            docker ps
            
        """
    }
}


        stage('Test image') {
            agent any
            steps {
                sh "curl http://172.17.0.1:8081/login"
            }
        }

        stage('Clean Container') {
            agent any
            steps {
                sh '''
                    docker stop paymybuddy || true
                    docker rm paymybuddy || true
                    docker stop mysql || true
                    docker rm mysql || true
                '''
            }
        }

        stage('Save Artefact') {
            agent any
            steps {
                sh '''
                    docker save ${CONTAINER_IMAGE} > /tmp/paymybuddy.tar
                    docker save mysqldb > /tmp/mysqldb.tar
                '''
            }
        }

        stage('Login and Push Image on Docker Hub') {
            agent any
            environment {
                DOCKERHUB = credentials('dockerhub')
            }
            steps {
                sh '''
                    docker login -u "${DOCKERHUB_USR}" --password-stdin ${DOCKERHUB_PSW}
                    docker push ${CONTAINER_IMAGE}
                    docker push ${IMAGE_MYSQL}
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
