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

        INTERNAL_PORT     = "5000"
        EXTERNAL_PORT     = "${PORT_EXPOSED}"
        CONTAINER_IMAGE   = "${ID_DOCKER}/${IMAGE_NAME}:${IMAGE_TAG}"
    }

    stages {

        stage('Java Version') {
            agent any
            steps {
                sh "java -version"
            }
        }

        stage('Unit Tests') {
            agent any
            steps {
                sh """
                    chmod +x mvnw
                    ./mvnw clean install
                    ./mvnw -B test
                """
            }
        }

        /*
        stage('SonarCloud') {
            agent any
            steps {
                withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
                    sh """
                        chmod +x mvnw
                        ./mvnw -B verify sonar:sonar \
                          -Dsonar.projectKey=kingsleyDeve_PayMyBuddy \
                          -Dsonar.organization=kingsleydeve \
                          -Dsonar.host.url=https://sonarcloud.io \
                          -Dsonar.login=${SONAR_TOKEN}
                    """
                }
            }
        }
        */

        stage('Test') {
            agent any
            steps {
                sh 'ls'
            }
        }

        stage('Run container based on built image') {
            agent any
            steps {
                sh """
                    curl -fsSL https://get.docker.com -o get-docker.sh
                    sh get-docker.sh

                    echo "Clean Environment"
                    docker rm -f ${IMAGE_NAME} || echo "container does not exist"

                    docker build -t ${ID_DOCKER}/${IMAGE_NAME}:${IMAGE_TAG} .

                    docker run --name ${IMAGE_NAME} -d -p ${EXTERNAL_PORT}:${INTERNAL_PORT} \
                        -e PORT=${INTERNAL_PORT} \
                        ${ID_DOCKER}/${IMAGE_NAME}:${IMAGE_TAG}

                    sleep 5
                """
            }
        }

        stage('Test image') {
            agent any
            steps {
                sh "curl http://172.17.0.1:${EXTERNAL_PORT}"
            }
        }

        stage('Clean Container') {
            agent any
            steps {
                sh """
                    docker stop ${IMAGE_NAME} || true
                    docker rm ${IMAGE_NAME} || true
                """
            }
        }

        stage('Save Artefact') {
            agent any
            steps {
                sh """
                    docker save ${ID_DOCKER}/${IMAGE_NAME}:${IMAGE_TAG} > /tmp/paymybuddy.tar
                """
            }
        }

        stage('Login and Push Image on Docker Hub') {
            agent any
            environment {
                DOCKERHUB_CREDS = credentials('dockerhub-credentials')
            }
            steps {
                sh """
                    echo "${DOCKERHUB_CREDS_PSW}" | docker login -u "${DOCKERHUB_CREDS_USR}" --password-stdin
                    docker push ${ID_DOCKER}/${IMAGE_NAME}:${IMAGE_TAG}
                """
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
            when { branch 'main' }
            agent any
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
                message: "kingsley - SUCCESS: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL}) - PROD => http://${PROD_APP_ENDPOINT} | STAGING => http://${STG_APP_ENDPOINT}"
            )
        }

        failure {
            slackSend(
                color: '#FF0000',
                message: "kingsley - FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL})"
            )
        }
    }
}
