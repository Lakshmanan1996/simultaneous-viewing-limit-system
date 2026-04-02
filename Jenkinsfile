pipeline {
    agent none

    environment {
        DOCKERHUB_USER = "lakshvar96"
        IMAGE = "viewing-limit-system"
        GIT_REPO = "https://github.com/Lakshmanan1996/simultaneous-viewing-limit-system.git"
    }

    stages {

        stage('Checkout Code') {
            agent { label 'workernode1' }
            steps {
                git branch: 'master', url: "${GIT_REPO}"
                stash includes: '**/*', name: 'source-code'
            }
        }

        stage('Build & SonarQube Analysis') {
            agent { label 'workernode2' }
            steps {
                unstash 'source-code'
                withSonarQubeEnv('sonarqube') {
                    sh """
                    mvn clean install sonar:sonar \
                    -Dsonar.projectKey=microservices
                    """
                }
            }
        }

        stage('Quality Gate') {
            agent { label 'workernode2' }
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

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

        stage('Trivy Scan') {
            agent { label 'workernode3' }
            steps {
                sh """
                trivy image --exit-code 1 --severity HIGH,CRITICAL ${DOCKERHUB_USER}/${IMAGE}:${BUILD_NUMBER}
                """
            }
        }

        stage('Push Image') {
            agent { label 'workernode3' }
            steps {
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
                docker system prune -f
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
