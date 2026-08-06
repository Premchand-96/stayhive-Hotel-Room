pipeline {

    agent any

    options {
        timestamps()
        ansiColor('xterm')
    }

    environment {

        FRONTEND = "frontend"
        BACKEND = "backend"

        FRONTEND_IMAGE = "guru0114/stayhive-frontend"
        BACKEND_IMAGE = "guru0114/stayhive-backend"

        IMAGE_TAG = "${BUILD_NUMBER}"

        SONAR_URL = "http://16.112.182.98:9000"

        NEXUS_URL = "16.112.182.98:8081"
        NEXUS_REPO = "stayhive-raw"
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout Source') {
            steps {
                checkout scm
            }
        }

        stage('Project Structure') {
            steps {
                sh '''
                pwd
                ls -la
                ls -la frontend
                ls -la backend
                '''
            }
        }

        stage('Install Backend Dependencies') {
            steps {
                dir("${BACKEND}") {
                    sh '''
                    npm install
                    '''
                }
            }
        }

        stage('Wait For SonarQube') {
            steps {
                sh '''
                echo "Checking SonarQube..."

                for i in {1..30}
                do
                    if curl -s ${SONAR_URL}/api/system/status >/dev/null
                    then
                        echo "SonarQube is reachable."
                        exit 0
                    fi

                    echo "Waiting for SonarQube..."
                    sleep 10
                done

                echo "SonarQube is not reachable."
                exit 1
                '''
            }
        }

        stage('SonarQube Scan') {
            steps {

                withCredentials([
                    string(credentialsId: 'sonar-token',
                           variable: 'SONAR_TOKEN')
                ]) {

                    sh '''
                    docker run --rm \
                      -e SONAR_HOST_URL=${SONAR_URL} \
                      -e SONAR_TOKEN=$SONAR_TOKEN \
                      -v "$WORKSPACE:/usr/src" \
                      -w /usr/src \
                      sonarsource/sonar-scanner-cli \
                      -Dsonar.projectKey=stayhive \
                      -Dsonar.projectName=StayHive \
                      -Dsonar.sources=. \
                      -Dsonar.exclusions=**/node_modules/**,**/.git/** \
                      -Dsonar.sourceEncoding=UTF-8
                    '''
                }
            }
        }

        stage('Docker Login') {

            steps {

                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {

                    sh '''
                    echo "$DOCKER_PASS" | docker login \
                    -u "$DOCKER_USER" \
                    --password-stdin
                    '''
                }
            }
        }

        stage('Remove Old Containers') {
            steps {
                sh '''
                docker rm -f stayhive-frontend || true
                docker rm -f stayhive-backend || true
                '''
            }
        }

        stage('Remove Old Images') {
            steps {
                sh '''
                docker rmi ${FRONTEND_IMAGE}:latest || true
                docker rmi ${BACKEND_IMAGE}:latest || true
                docker rmi ${FRONTEND_IMAGE}:${IMAGE_TAG} || true
                docker rmi ${BACKEND_IMAGE}:${IMAGE_TAG} || true
                docker image prune -af || true
                '''
            }
        }

        stage('Build Docker Images') {

            steps {

                sh """
                docker build --no-cache \
                -t ${FRONTEND_IMAGE}:${IMAGE_TAG} \
                ./frontend

                docker build --no-cache \
                -t ${BACKEND_IMAGE}:${IMAGE_TAG} \
                ./backend

                docker tag ${FRONTEND_IMAGE}:${IMAGE_TAG} ${FRONTEND_IMAGE}:latest
                docker tag ${BACKEND_IMAGE}:${IMAGE_TAG} ${BACKEND_IMAGE}:latest
                """
            }
        }

        stage('Push Docker Images') {

            steps {

                sh """
                docker push ${FRONTEND_IMAGE}:${IMAGE_TAG}
                docker push ${FRONTEND_IMAGE}:latest

                docker push ${BACKEND_IMAGE}:${IMAGE_TAG}
                docker push ${BACKEND_IMAGE}:latest
                """
            }
        }

        stage('Create ZIP Artifact') {
            steps {

                sh '''
                rm -f stayhive.zip

                zip -r stayhive.zip . \
                -x "*.git*" \
                -x "backend/node_modules/*" \
                -x "stayhive.zip"

                ls -lh stayhive.zip
                '''
            }
        }

        stage('Upload To Nexus') {

            steps {

                nexusArtifactUploader(

                    nexusVersion: 'nexus3',

                    protocol: 'http',

                    nexusUrl: "${NEXUS_URL}",

                    repository: "${NEXUS_REPO}",

                    groupId: 'com.stayhive',

                    version: "1.0.${BUILD_NUMBER}",

                    credentialsId: 'nexus',

                    artifacts: [[

                        artifactId: 'stayhive',

                        classifier: '',

                        file: 'stayhive.zip',

                        type: 'zip'

                    ]]
                )
            }
        }

        stage('Deploy Using Docker Compose') {

            steps {

                sh '''
                docker compose down || true

                docker compose pull

                docker compose up -d --force-recreate

                docker image prune -af
                '''
            }
        }

        stage('Health Check') {

            steps {

                sh '''
                echo "Containers"

                docker ps

                echo ""

                docker compose ps
                '''
            }
        }

    }

    post {

        success {

            echo "================================="
            echo "StayHive Deployment Successful"
            echo "================================="
        }

        failure {

            echo "================================="
            echo "StayHive Deployment Failed"
            echo "================================="
        }

        always {

            cleanWs()
        }

    }

}
