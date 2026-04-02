pipeline {
    agent none

    environment {
        DOCKERHUB_USER = "lakshvar96"
        IMAGE = "viewing-limit-system"
        GIT_REPO = "https://github.com/Lakshmanan1996/simultaneous-viewing-limit-system.git"
    }

    stages {

        /* ===================== CHECKOUT ===================== */
        stage('Checkout Code') {
            agent{ label 'workernode1' }
            steps {
                git branch: 'main', url: "${GIT_REPO}"
            }
        }

        /* ===================== STASH SOURCE ===================== */
        stage('Stash Source') {
            agent { label 'workernode1' }
            steps {
                stash includes: '**/*', name: 'source-code'
            }
        }

        /* ===================== BUILD ===================== */
        stage('Build') {
            agent { label 'workernode1' }
            steps {
                unstash 'source-code'
                sh '''
                # If NodeJS project
                if [ -f package.json ]; then
                    npm install
                fi

                # If Maven project
                if [ -f pom.xml ]; then
                    mvn clean package
                fi
                '''
            }
        }

        /* ===================== SONARQUBE ===================== */
        stage('SonarQube Analysis') {
            agent { label 'workernode2' }
            steps {
                unstash 'source-code'

                script {
                    def scannerHome = tool 'SonarQubeScanner'
                    withSonarQubeEnv('sonarqube') {
                        sh """
                        ${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=microservices \
                        -Dsonar.sources=.
                        """
                    }
                }
            }
        }

        /* ===================== QUALITY GATE ===================== */
        stage('Quality Gate') {
            agent { label 'workernode2' }
            steps { 
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        /* ===================== DOCKER BUILD ===================== */
        stage('Docker Build') {
            agent { label 'workernode3' }
            steps {
                unstash 'source-code'

                sh """
                docker build -t ${DOCKERHUB_USER}/${IMAGE}:${BUILD_NUMBER} .
                docker tag ${DOCKERHUB_USER}/${IMAGE}:${BUILD_NUMBER} ${DOCKERHUB_USER}/${IMAGE}:latest
                """
            }
        }

        /* ===================== TRIVY SCAN ===================== */
        stage('Trivy Scan') {
            agent { label 'workernode3' }
            steps {
                

                sh """
                trivy image --exit-code 1 --severity HIGH,CRITICAL ${DOCKERHUB_USER}/${IMAGE}:${BUILD_NUMBER}
                """
            }
        }

        /* ===================== PUSH IMAGE ===================== */
        stage('Push Image') {
            agent { label 'workernode3' }
            steps {
                unstash 'source-code'

                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh 'echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin'
                }

                sh """
                docker push ${DOCKERHUB_USER}/${IMAGE}:${BUILD_NUMBER}
                docker push ${DOCKERHUB_USER}/${IMAGE}:latest
                """
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline SUCCESS"
        }
        failure {
            echo "❌ Pipeline FAILED"
        }
    }
}
