@Library("test")_

pipeline {

    agent none

    environment {
        
        IMAGE_NAME       = "paymybuddy"
        IMAGE_TAG        = "latest"
        IMAGE_MYSQL      = "paymybuddy-db"
        STAGING_SERVER   = "52.47.108.120"
        PROD_SERVER      = "13.39.105.27"
        DEPLOY_USER      = "ubuntu"

        CONTAINER_IMAGE        = "kingsley95/${IMAGE_NAME}:${IMAGE_TAG}"
        MYSQL_CONTAINER_IMAGE  = "kingsley95/${IMAGE_MYSQL}:${IMAGE_TAG}"
    }

    stages {

        /* ---------------- UNIT TESTS ---------------- */
        stage('Unit Tests') {
            agent any
            steps {
                sh """
                    chmod +x mvnw
                    ./mvnw clean install -DskipTests
                    ./mvnw -B test
                """
            }
        }

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


        

        /* ---------------- BUILD DOCKER ---------------- */
        stage('Build & Run Docker Image') {
            agent any
            steps {
                sh """
                    docker stop ${IMAGE_NAME} || true
                    docker rm -f ${IMAGE_NAME} || true
                    docker stop mysql || true
                    docker rm mysql || true

                    docker network create paymybuddy-net || true

                    docker build -t ${CONTAINER_IMAGE} .
                    docker build -f Dockerfile-db -t ${IMAGE_MYSQL} .

                    docker run --name mysql --network paymybuddy-net \
                        -e MYSQL_ROOT_PASSWORD=password \
                        -e MYSQL_PASSWORD=password \
                        -e MYSQL_USER=test \
                        -e MYSQL_DATABASE=db_paymybuddy \
                        -p 3306:3306 -d ${IMAGE_MYSQL}

                    sleep 30

                    docker run --name ${IMAGE_NAME} \
                        --network paymybuddy-net \
                        -p 8081:8080 \
                        --link mysql \
                        -d ${CONTAINER_IMAGE}

                    sleep 10
                    docker ps
                """
            }
        }

        /* ---------------- TEST IMAGE ---------------- */
        stage('Test image') {
            agent any
            steps {
                sh "curl http://172.17.0.1:8081/login"
            }
        }

        /* ---------------- CLEAN ---------------- */
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

        /* ---------------- SAVE ARTEFACTS ---------------- */
        stage('Save Artefact') {
            agent any
            steps {
                sh '''
                    docker save ${CONTAINER_IMAGE} > /tmp/paymybuddy.tar
                    docker save ${MYSQL_CONTAINER_IMAGE} > /tmp/mysqldb.tar
                '''
            }
        }

        /* ---------------- PUSH DOCKERHUB ---------------- */
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

        stage('Deploy Staging') {
            agent any
            when {
                expression { env.GIT_BRANCH != "origin/main" }
            }
            steps {
                script {
                    echo "Déploiement en STAGING sur ${STAGING_SERVER}..."
                    deployServer(STAGING_SERVER)

                    echo "Validation STAGING..."
                    sh """
                        until curl  http://${STAGING_SERVER}:8080/login; do
                            echo "En attente du démarrage de l'application..."
                            sleep 5
                        done
                    """
                }
            }
        }

        /* ---------------- DEPLOY PRODUCTION ---------------- */
        stage('Deploy Production') {
            agent any
            when {
                expression { env.GIT_BRANCH == "origin/main" }
            }
            steps {
                script {
                    echo "Déploiement en PRODUCTION sur ${PROD_SERVER}..."
                    deployServer(PROD_SERVER)

                    echo "Validation PRODUCTION..."
                    sh """
                            curl  http://${PROD_SERVER}:8080/login
                            echo "production deployed"
                           
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
} // ← fin du pipeline


/* --------------------------------------------------------
   Fonction globale de déploiement
-------------------------------------------------------- */
def deployServer(String server) {
    sshagent (credentials: ['deployapp']) {
        sh """
            ssh -o StrictHostKeyChecking=no ${DEPLOY_USER}@${server} "curl -fsSL https://get.docker.com | sh"

            ssh -o StrictHostKeyChecking=no ${DEPLOY_USER}@${server} "sudo docker pull ${CONTAINER_IMAGE}"
            ssh -o StrictHostKeyChecking=no ${DEPLOY_USER}@${server} "sudo docker pull ${MYSQL_CONTAINER_IMAGE}"

            ssh -o StrictHostKeyChecking=no ${DEPLOY_USER}@${server} \
                "sudo docker stop paymybuddy || true && sudo docker rm paymybuddy || true"

            ssh -o StrictHostKeyChecking=no ${DEPLOY_USER}@${server} \
                "sudo docker stop mysql || true && sudo docker rm mysql || true"

             ssh -o StrictHostKeyChecking=no ${DEPLOY_USER}@${server} \
                 "sudo docker network rm paymybuddy-net || true"

                 ssh -o StrictHostKeyChecking=no ${DEPLOY_USER}@${server} \
                 "sudo docker network create paymybuddy-net || true"

                 ssh -o StrictHostKeyChecking=no ${DEPLOY_USER}@${server} \
                "sudo docker run -d --name mysql  --network paymybuddy-net \
                        -e MYSQL_ROOT_PASSWORD=password \
                        -e MYSQL_PASSWORD=password \
                        -e MYSQL_USER=test \
                        -e MYSQL_DATABASE=db_paymybuddy -p 3306:3306 ${MYSQL_CONTAINER_IMAGE}"
            
            ssh -o StrictHostKeyChecking=no ${DEPLOY_USER}@${server} \
            "sleep 30"
            
            ssh -o StrictHostKeyChecking=no ${DEPLOY_USER}@${server} \
                "sudo docker run -d --name paymybuddy -p 8080:8080 ${CONTAINER_IMAGE}" 

                ssh -o StrictHostKeyChecking=no ${DEPLOY_USER}@${server} \
            "sleep 20"
        """
    }
}
