pipeline {

    agent none

    environment {
        ID_DOCKER        = "${ID_DOCKER_PARAMS}"
        IMAGE_NAME       = "paymybuddy"
        IMAGE_TAG        = "latest"
        APP_NAME         = "kingsley"
        IMAGE_MYSQL      = "paymybuddy-db"
        STAGING_SERVER = "13.220.199.237"
        PROD_SERVER = "98.81.19.212"
        DEPLOY_USER = "ubuntu"
        
       
        CONTAINER_IMAGE   = "kingsley95/${IMAGE_NAME}:${IMAGE_TAG}"
        MYSQL_CONTAINER_IMAGE   = "kingsley95/${IMAGE_MYSQL}:${IMAGE_TAG}"
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
                    echo "${DOCKERHUB_PSW}" | docker login -u "${DOCKERHUB_USR}" --password-stdin 
                    docker push ${CONTAINER_IMAGE}
                    docker push ${MYSQL_CONTAINER_IMAGE}
                '''
            }
        }
stages {

        /* ---------------- STAGING ---------------- */
        stage('Deploy Staging') {
            when {
                not { branch 'main' }   // s’exécute pour toutes les branches sauf main
            }
            steps {
                script {
                    echo "Déploiement en STAGING sur ${STAGING_SERVER}..."
                    deployServer(STAGING_SERVER)

                    echo "Validation STAGING..."
                    sh """
                        until curl -sf http://${STAGING_SERVER}:8080/actuator/health; do
                            echo "En attente du démarrage de l'application..."
                            sleep 5
                        done
                    """
                }
            }
        }

        /* ---------------- PRODUCTION ---------------- */
        stage('Deploy Production') {
            when {
                branch 'main'   // ne s’exécute QUE sur main
            }
            steps {
                script {
                    echo "Déploiement en PRODUCTION sur ${PROD_SERVER}..."
                    deployServer(PROD_SERVER)

                    echo "Validation PRODUCTION..."
                    sh """
                        until curl -sf http://${PROD_SERVER}:8080/actuator/health; do
                            echo "En attente du démarrage..."
                            sleep 5
                        done
                    """
                }
            }
        }
    }

    post {
        success {
            slackSend(
                color: "good",
                message: "SUCCESS : Pipeline OK pour ${env.BRANCH_NAME} (#${env.BUILD_NUMBER})"
            )
        }
        failure {
            slackSend(
                color: "danger",
                message: "FAILURE : Pipeline échoué pour ${env.BRANCH_NAME} (#${env.BUILD_NUMBER})"
            )
        }
    }
}


/* --------------------------------------------------------
   Fonction globale de déploiement
-------------------------------------------------------- */
def deployServer(String server) {
    sshagent (credentials: ['deployapp']) {
        sh """
            ssh -o StrictHostKeyChecking=no ${DEPLOY_USER}@${server} "curl -fsSL https://get.docker.com | sh"
            
            ssh -o StrictHostKeyChecking=no ${DEPLOY_USER}@${server} "docker pull $CONTAINER_IMAGE"
            
            ssh -o StrictHostKeyChecking=no ${DEPLOY_USER}@${server} "docker pull $MYSQL_CONTAINER_IMAGE"

            ssh -o StrictHostKeyChecking=no ${DEPLOY_USER}@${server} \
                "docker stop paymybuddy || true && docker rm paymybuddy || true"
                
            ssh -o StrictHostKeyChecking=no ${DEPLOY_USER}@${server} \
                "docker stop mysql || true && docker rm mysql || true"
                
            ssh -o StrictHostKeyChecking=no ${DEPLOY_USER}@${server} \
                "docker run -d --name mysql -p 8080:8080 ${CONTAINER_IMAGE}"
                
            ssh -o StrictHostKeyChecking=no ${DEPLOY_USER}@${server} \
                "docker run -d --name mysql -p 3306:3306 ${MYSQL_CONTAINER_IMAGE}"
                
        """
    }
}
}
