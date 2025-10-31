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

        REVIEW_SERVER = '34.224.99.183'
        STAGING_SERVER = '54.86.117.55'
        PROD_SERVER = '3.88.29.115'

        DEPLOY_USER = 'ubuntu'
    }

    options {
        timestamps()
    }

    stages {
        stage('Install Docker CLI') {
            steps {
                echo "Installation de Docker CLI..."
                sh '''
                    apt-get update -o Acquire::Check-Valid-Until=false -o Acquire::Check-Date=false
                    apt-get install -y docker.io
                    docker --version
                '''
            }
        }

        stage('Checkout') {
            steps {
                checkout scm
                echo "Code source récupéré ✅"
            }
        }

        stage('Tests Automatisés') {
            steps {
                sh 'mvn clean verify'
            }
        }

        stage('Analyse SonarCloud') {
            steps {
                withSonarQubeEnv('SonarCloud') {
                    sh '''
                        mvn sonar:sonar \
                          -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                          -Dsonar.organization=${SONAR_ORG} \
                          -Dsonar.host.url=https://sonarcloud.io \
                          -Dsonar.login=${SONAR_TOKEN}
                    '''
                }
            }
        }

        stage('Quality Gate') {
            when { expression { false } } // ignoré
            steps {
                echo "Stage Quality Gate ignoré"
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
                script {
                    sh '''
                        docker login -u ${DOCKERHUB_CREDENTIALS_USR} -p ${DOCKERHUB_CREDENTIALS_PSW}
                        docker build -t ${DOCKER_IMAGE} .
                        docker push ${DOCKER_IMAGE}
                    '''
                }
            }
        }
    // --- Étape Review (uniquement pour les branches ≠ main) ---
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

        // --- Étape Staging (uniquement sur main) ---
        stage('Déploiement Staging') {
            when {
                branch 'main'
            }
            steps {
                echo " Déploiement en pré-production (${STAGING_SERVER})..."
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

        // --- Étape Production ---
        stage('Déploiement Production') {
            when {
                branch 'main'
            }
            steps {
                echo " Déploiement final en production (${PROD_SERVER})..."
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

    }

    post {
        success {
            echo "✅ Pipeline réussie !"
            slackSend(channel: env.SLACK_CHANNEL, message: "✅ Pipeline réussie pour ${env.BRANCH_NAME} (#${env.BUILD_NUMBER})")
        }
        failure {
            echo "❌ Pipeline échouée."
            slackSend(channel: env.SLACK_CHANNEL, message: "❌ Pipeline échouée pour ${env.BRANCH_NAME} (#${env.BUILD_NUMBER})")
        }
        always {
            echo "Fin du pipeline Jenkins."
        }
    }

