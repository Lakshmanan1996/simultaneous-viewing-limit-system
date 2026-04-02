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
            }
        }

        stage('Stash Source') {
            agent { label 'workernode1' }
            steps {
                stash includes: '**/*', name: 'source-code'
            }
        }

        stage('Build') {
            agent { label 'workernode1' }
            steps {
                unstash 'source-code'
                sh '''
                if [ -f package.json ]; then
                    npm install
                fi

                if [ -f pom.xml ]; then
                    mvn clean package
                fi
                '''
            }
        }

        stage('SonarQube Analysis') {
            agent { label 'workernode2' }
            steps {
                unstash 'source-code'
                sh "mvn clean install -DskipTests" 
                
                script {
                    def scannerHome = tool 'SonarQubeScanner'
                    withSonarQubeEnv('sonarqube') {
                        sh """
                        ${scannerHome}/bin/sonar-scanner \
                        -Dsonar.projectKey=microservices \
                        -Dsonar.sources=check-service/src,push-service/src \
                        -Dsonar.java.binaries=check-service/target,push-service/target
                        """
                    }
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
