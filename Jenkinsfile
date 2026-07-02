pipeline {
    agent any

    environment {
        // IDs des credentials stockés dans Jenkins
        DOCKERHUB_CREDENTIALS = credentials('maxime-dockerhub-credentials-id') 
        SONAR_TOKEN           = credentials('maxime-backend-sonarqube-token-id')
        
        SONAR_HOST_URL       = 'https://sonarqube.cicd.kits.ext.educentre.fr'
        IMAGE_NAME           = 'mdrmgn/maxime-tasklist-backend-eval'
        IMAGE_TAG            = 'latest'
    }

    triggers {
        pollSCM('H/5 * * * *') 
    }

    stages {
        stage('Installation des dépendances') {
            steps {
                sh 'npm ci'
                sh 'npx prisma generate'
            }
        }

        stage('Exécution des tests & Couverture') {
            steps {
                sh 'npx prisma generate --schema prisma/schema-test.prisma'
                sh 'npm run test:coverage'
                sh 'npm run test:e2e:coverage'
            }
        }

        stage('Analyse SonarQube') {
            steps {
                sh "npm install -g sonarqube-scanner"
                sh """
                    sonar-scanner \
                    -Dsonar.host.url=${SONAR_HOST_URL} \
                    -Dsonar.token=${SONAR_TOKEN}
                """
            }
        }

        stage('Construction Docker (Local)') {
            steps {
                sh 'docker buildx build --tag maxime-tasklist-backend:local --load .'
            }
        }

        stage('Scan de Sécurité (Trivy)') {
            steps {
                // Ajout de --cache-dir pour éviter les conflits de lock globaux
                sh 'trivy image --cache-dir .trivycache/ --severity CRITICAL,HIGH --format table maxime-tasklist-backend:local'
            }
        }

        stage('Génération des SBOM') {
            steps {
                // Ajout de --cache-dir également ici
                sh 'trivy image --cache-dir .trivycache/ --format spdx-json --output sbom-spdx.json maxime-tasklist-backend:local'
                sh 'trivy image --cache-dir .trivycache/ --format cyclonedx --output sbom-cyclonedx.json maxime-tasklist-backend:local'
            }
        }

        stage('Publication sur DockerHub') {
            steps {
                sh "echo ${DOCKERHUB_CREDENTIALS_PSW} | docker login -u ${DOCKERHUB_CREDENTIALS_USR} --password-stdin"
                sh """docker buildx build --platform linux/amd64 --tag ${IMAGE_NAME}:${IMAGE_TAG} --sbom true --provenance=true --push ."""
            }
        }
    }

    post {
        always {
            cleanWs()
        }
    }
}