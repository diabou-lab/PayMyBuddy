pipeline {
    agent {
        docker {
            // Image Maven + JDK 17 + Docker CLI intégré
            image 'thecodingmachine/maven:3.9.6-eclipse-temurin-17-docker'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }

    environment {
        // Credentials Jenkins
        SONAR_TOKEN = credentials('sonarcloud-token')
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
        SSH_KEY = credentials('deployapp')
        SLACK_CHANNEL = '#jenkins-eazytrainingdiabou'

        // Configuration Sonar & Docker
        SONAR_PROJECT_KEY = 'diabou-lab_PayMyBuddy'
        SONAR_ORG = 'diabou-lab'
        APP_NAME = 'springboot-app'
        DOCKER_IMAGE = "diaboudoc/${APP_NAME}:${env.BUILD_NUMBER}"

        // Serveurs
        REVIEW_SERVER = '34.224.99.183'
        STAGING_SERVER = '54.86.117.55'
        PROD_SERVER = '3.88.29.115'

        DEPLOY_USER = 'ubuntu'
    }

    options {
        timestamps()
    }

    stages {
        stage('Checkout') {
            steps {
                echo "Récupération du code source..."
                checkout scm
            }
        }

        stage('Tests Automatisés') {
            steps {
                echo "Exécution des tests unitaires et d'intégration..."
                sh 'mvn clean verify'
            }
        }

        stage('Analyse SonarCloud') {
            steps {
                withSonarQubeEnv('SonarCloud') {
                    sh '''
                        mvn clean verify sonar:sonar \
                          -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                          -Dsonar.organization=${SONAR_ORG} \
                          -Dsonar.host.url=https://sonarcloud.io \
                          -Dsonar.login=${SONAR_TOKEN} \
                          -Dsonar.projectVersion=${BUILD_NUMBER} \
                          -Dsonar.analysis.buildNumber=${BUILD_NUMBER}
                    '''
                }
            }
        }

        stage('Quality Gate') {
            when {
                expression { false } // Ignoré
            }
            steps {
                echo "Stage Quality Gate ignoré."
            }
        }

        stage('Compilation & Packaging') {
            steps {
                script {
                    catchError(buildResult: 'UNSTABLE', stageResult: 'SUCCESS') {
                        sh 'mvn clean package -DskipTests'
                    }
                }
            }
        }

        stage('Build & Push Docker Image') {
            steps {
                echo "Construction et envoi de l’image DockerHub..."
                script {
                    sh """
                        docker login -u ${DOCKERHUB_CREDENTIALS_USR} -p ${DOCKERHUB_CREDENTIALS_PSW}
                        docker build -t ${DOCKER_IMAGE} .
                        docker push ${DOCKER_IMAGE}
                    """
                }
            }
        }

        stage('Déploiement Review') {
            when {
                not { branch 'main' }
            }
            steps {
                echo "Déploiement en environnement Review (${REVIEW_SERVER})..."
                sshagent (credentials: ['deployapp']) {
                    sh """
                        ssh ${DEPLOY_USER}@${REVIEW_SERVER} "which docker || (curl -fsSL https://get.docker.com | sh)"
                        ssh ${DEPLOY_USER}@${REVIEW_SERVER} "docker pull ${DOCKER_IMAGE}"
                        ssh ${DEPLOY_USER}@${REVIEW_SERVER} "docker stop review-app || true && docker rm review-app || true"
                        ssh ${DEPLOY_USER}@${REVIEW_SERVER} "docker run -d --name review-app -p 8082:8080 ${DOCKER_IMAGE}"
                    """
                }
            }
        }

        stage('Déploiement Staging') {
            when {
                branch 'main'
            }
            steps {
                echo "Déploiement en pré-production (${STAGING_SERVER})..."
                sshagent (credentials: ['deployapp']) {
                    sh """
                        ssh ${DEPLOY_USER}@${STAGING_SERVER} "which docker || (curl -fsSL https://get.docker.com | sh)"
                        ssh ${DEPLOY_USER}@${STAGING_SERVER} "docker pull ${DOCKER_IMAGE}"
                        ssh ${DEPLOY_USER}@${STAGING_SERVER} "docker stop staging-app || true && docker rm staging-app || true"
                        ssh ${DEPLOY_USER}@${STAGING_SERVER} "docker run -d --name staging-app -p 8081:8080 ${DOCKER_IMAGE}"
                    """
                }
            }
        }

        stage('Validation Staging') {
            when {
                branch 'main'
            }
            steps {
                echo "Vérification du déploiement en staging..."
                sh "curl -f http://${STAGING_SERVER}:8081/actuator/health"
            }
        }

        stage('Déploiement Production') {
            when {
                branch 'main'
            }
            steps {
                echo "Déploiement final en production (${PROD_SERVER})..."
                sshagent (credentials: ['deployapp']) {
                    sh """
                        ssh ${DEPLOY_USER}@${PROD_SERVER} "which docker || (curl -fsSL https://get.docker.com | sh)"
                        ssh ${DEPLOY_USER}@${PROD_SERVER} "docker pull ${DOCKER_IMAGE}"
                        ssh ${DEPLOY_USER}@${PROD_SERVER} "docker stop prod-app || true && docker rm prod-app || true"
                        ssh ${DEPLOY_USER}@${PROD_SERVER} "docker run -d --name prod-app -p 8080:8080 ${DOCKER_IMAGE}"
                    """
                }
            }
        }

        stage('Validation Production') {
            when {
                branch 'main'
            }
            steps {
                echo "Vérification du déploiement en production..."
                sh "curl -f http://${PROD_SERVER}:8080/actuator/health"
            }
        }
    }

    post {
        success {
            echo "Pipeline réussie "
            slackSend(channel: "${SLACK_CHANNEL}", message: "Pipeline réussie pour ${env.BRANCH_NAME} (#${env.BUILD_NUMBER})")
        }
        failure {
            echo "❌ Échec du pipeline."
            slackSend(channel: "${SLACK_CHANNEL}", message: "❌ Pipeline échouée pour ${env.BRANCH_NAME} (#${env.BUILD_NUMBER})")
        }
        always {
            echo "Fin du pipeline Jenkins."
        }
    }
}
