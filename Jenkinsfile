pipeline {
    agent {
        docker {
            image 'maven:3.9.6-eclipse-temurin-17'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }

    environment {
        ID_DOCKER = "${ID_DOCKER_PARAMS}"
        IMAGE_NAME = "paymybuddy"
        IMAGE_TAG = "latest"
        APP_NAME = "kingsley"

        STG_API_ENDPOINT = "ip10-0-57-8-d4bogugltosglhl3v92g-1993.direct.docker.labs.eazytraining.fr"
        STG_APP_ENDPOINT = "ip10-0-57-8-d4bogugltosglhl3v92g-80.direct.docker.labs.eazytraining.fr"
        PROD_API_ENDPOINT = "ip10-0-57-9-d4bogugltosglhl3v92g-1993.direct.docker.labs.eazytraining.fr"
        PROD_APP_ENDPOINT = "ip10-0-57-9-d4bogugltosglhl3v92g-80.direct.docker.labs.eazytraining.fr"

        INTERNAL_PORT = "5000"
        EXTERNAL_PORT = "${PORT_EXPOSED}"
        CONTAINER_IMAGE = "${ID_DOCKER}/${IMAGE_NAME}:${IMAGE_TAG}"
    }
   
    stages {




        
        stage('Build') {
            steps {
                // on utilise mvnw pour s'assurer de la version embarquée
                sh './mvnw -B clean package -DskipTests'
            }
        }

        stage('Unit Tests') {
            steps {
                sh './mvnw -B test'
            }
            post {
                always {
                    // publication des résultats JUnit
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('Test') {
          agent any
          steps {
                sh 'ls'
            }
        }

        stage('SonarCloud') {
            steps {
                sh 'mvn test -X'
            }
        }

        stage('Run container based on built image') {
            agent any
            steps {
                sh '''
                    echo "Clean Environment"
                    docker rm -f $IMAGE_NAME || echo "container does not exist"
                    docker run --name $IMAGE_NAME -d -p ${PORT_EXPOSED}:${INTERNAL_PORT} \
                    -e PORT=${INTERNAL_PORT} ${ID_DOCKER}/$IMAGE_NAME:$IMAGE_TAG
                    sleep 5
                '''
            }
        }

        stage('Test image') {
            agent any
            steps {
                sh 'curl http://172.17.0.1:${PORT_EXPOSED}'
            }
        }

        stage('Clean Container') {
            agent any
            steps {
                sh '''
                    docker stop $IMAGE_NAME
                    docker rm $IMAGE_NAME
                '''
            }
        }

        stage('Save Artefact') {
            agent any
            steps {
                sh '''
                    docker save ${ID_DOCKER}/$IMAGE_NAME:$IMAGE_TAG > /tmp/alpinehelloworld.tar
                '''
            }
        }

        stage('Login and Push Image on Docker Hub') {
            agent any
            environment {
                DOCKERHUB_PASSWORD = credentials('dockerhub-credentials')
            }
            steps {
                sh '''
                    echo $DOCKERHUB_PASSWORD_PSW | docker login -u $ID_DOCKER --password-stdin
                    docker push ${ID_DOCKER}/$IMAGE_NAME:$IMAGE_TAG
                '''
            }
        }

        stage('STAGING - Deploy app') {
            agent any
            steps {
                sh """
                    echo '{\\"your_name\\":\\"${APP_NAME}\\",\\"container_image\\":\\"${CONTAINER_IMAGE}\\", \\"external_port\\":\\"${EXTERNAL_PORT}\\", \\"internal_port\\":\\"${INTERNAL_PORT}\\"}' > data.json 
                    curl -X POST http://${STG_API_ENDPOINT}/staging -H 'Content-Type: application/json' --data-binary @data.json 
                """
            }
        }

        stage('PRODUCTION - Deploy app') {
            when { expression { env.GIT_BRANCH == 'origin/main' } }
            agent any
            steps {
                sh """
                    curl -X POST http://${PROD_API_ENDPOINT}/prod -H 'Content-Type: application/json' \
                    -d '{"your_name":"${APP_NAME}","container_image":"${CONTAINER_IMAGE}", "external_port":"${EXTERNAL_PORT}", "internal_port":"${INTERNAL_PORT}"}'
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
                message: "kingsley - SUCCESSFUL: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]' (${env.BUILD_URL}) - PROD URL => http://${PROD_APP_ENDPOINT} , STAGING URL => http://${STG_APP_ENDPOINT}"
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
