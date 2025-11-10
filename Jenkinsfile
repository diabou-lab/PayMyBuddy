pipeline {
    agent {
        docker {
            image 'maven:3.9.6-eclipse-temurin-17'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }

    environment {
        SONAR_TOKEN = credentials('sonarcloud-token')
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
        SSH_KEY = credentials('deployapp')
        SLACK_CHANNEL = '#jenkins-eazytrainingdiabou'
        SONAR_PROJECT_KEY = 'diabou-lab_PayMyBuddy'
        SONAR_ORG = 'diabou-lab'
        APP_NAME = 'springboot-app'
        DOCKER_IMAGE = "diaboudoc/${APP_NAME}:${env.BUILD_NUMBER}"

        REVIEW_SERVER = '98.94.6.240'
        STAGING_SERVER = '54.197.32.145'
        PROD_SERVER = '34.224.40.53'

        DEPLOY_USER = 'ubuntu'
    }

    options { timestamps() }

    stages {
        stage('Pipeline Principal') {
            steps {
                script {
                    try {
                        // --- Installation Docker CLI ---
                        echo "Installation de Docker CLI..."
                        sh '''
                            apt-get update -o Acquire::Check-Valid-Until=false -o Acquire::Check-Date=false
                            apt-get install -y docker.io
                            docker --version
                        '''

                        // --- Checkout ---
                        checkout scm
                        echo "Code source récupéré "

                        // --- Tests Automatisés ---
                        sh 'mvn clean verify'

                        // --- Analyse SonarCloud ---
                        withSonarQubeEnv('SonarCloud') {
                            sh """
                                mvn sonar:sonar \
                                  -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                                  -Dsonar.organization=${SONAR_ORG} \
                                  -Dsonar.host.url=https://sonarcloud.io \
                                  -Dsonar.login=${SONAR_TOKEN}
                            """
                        }

                        // --- Compilation & Packaging ---
                        catchError(buildResult: 'UNSTABLE', stageResult: 'SUCCESS') {
                            sh 'mvn clean package -DskipTests'
                        }

                        // --- Build & Push Docker Image ---
                        sh """
                            docker login -u ${DOCKERHUB_CREDENTIALS_USR} -p ${DOCKERHUB_CREDENTIALS_PSW}
                            docker build -t ${DOCKER_IMAGE} .
                            docker push ${DOCKER_IMAGE}
                        """

                        // --- Déploiements selon les branches ---
                        sh 'apt-get install -y openssh-client'

                        if (env.BRANCH_NAME != 'main') {
                            echo "Déploiement Review..."
                            deployServer(env.REVIEW_SERVER, 'review-app', 8082)
                        } else {
                            echo "Déploiement Staging..."
                            deployServer(env.STAGING_SERVER, 'staging-app', 8081)
                            echo "Validation Staging..."
                            sh """
                                until curl -sf http://${STAGING_SERVER}:8081/actuator/health; do
                                    echo "En attente du démarrage..."
                                    sleep 5
                                done
                            """

                            echo "Déploiement Production..."
                            deployServer(env.PROD_SERVER, 'prod-app', 8080)
                            echo "Validation Production..."
                            sh """
                                until curl -sf http://${PROD_SERVER}:8080/actuator/health; do
                                    echo "En attente du démarrage..."
                                    sleep 5
                                done
                            """
                        }

                        // --- Notification Slack succès ---
                        slackSend(
                            channel: env.SLACK_CHANNEL,
                            color: 'good',   // vert
                            message: "Pipeline réussie pour ${env.BRANCH_NAME} (#${env.BUILD_NUMBER})"
                        )

                    } catch (Exception e) {
                        // --- Notification Slack échec ---
                        slackSend(
                            channel: env.SLACK_CHANNEL,
                            color: 'danger', // rouge
                            message: "Pipeline échouée pour ${env.BRANCH_NAME} (#${env.BUILD_NUMBER})\nErreur: ${e}"
                        )
                        error("Pipeline échouée: ${e}")
                    } finally {
                        echo "Fin du pipeline Jenkins."
                    }
                }
            }
        }
    }
}

// --- Fonctions globales ---
def deployServer(String server, String containerName, int port) {
    sshagent (credentials: ['deployapp']) {
        sh """
        ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${DEPLOY_USER}@${server} "command -v docker || (curl -fsSL https://get.docker.com | sh)"
        ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${DEPLOY_USER}@${server} "docker pull ${DOCKER_IMAGE}"
     #   ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${DEPLOY_USER}@${server} "docker stop ${containerName} || true && docker rm ${containerName} || true"
        ssh -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null ${DEPLOY_USER}@${server} "docker run -d --name ${containerName} -p ${port}:8080 ${DOCKER_IMAGE}"
        """
    }
}
