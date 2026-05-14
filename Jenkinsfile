pipeline {

    agent none

    tools {
        maven 'Maven'
        jdk 'JDK17'
        
    }

    environment {

        IMAGE1       = "check-service"
        IMAGE2       = "push-service"
        DOCKERHUB_USER = "lakshvar96"
        GIT_REPO = "https://github.com/kimyuuum/simultaneous-viewing-limit-system.git"
    }
    
   /* =====================================================   
   CHECKOUT
    ===================================================== */

    stages {

        stage('Checkout Code') {
            agent { label 'workernode1' }
            steps {
                checkout([$class: 'GitSCM',
                    branches: [[name: 'master']],
                    userRemoteConfigs: [[url: "${GIT_REPO}"]]
                ])
            }
        }


        stage('Stash Source') {
            agent { label 'workernode1' }
            steps {
                stash includes: '**/*', name: 'source-code'
            }
        }


        /* ===================== Build Maven Stage ===================== */
        stage('Build') {
            agent { label 'workernode2' }
            
            environment {
                 MAVEN_OPTS = '''
                --add-opens=jdk.compiler/com.sun.tools.javac.api=ALL-UNNAMED
                --add-opens=jdk.compiler/com.sun.tools.javac.code=ALL-UNNAMED
                --add-opens=jdk.compiler/com.sun.tools.javac.comp=ALL-UNNAMED
                --add-opens=jdk.compiler/com.sun.tools.javac.file=ALL-UNNAMED
                --add-opens=jdk.compiler/com.sun.tools.javac.main=ALL-UNNAMED
                --add-opens=jdk.compiler/com.sun.tools.javac.model=ALL-UNNAMED
                --add-opens=jdk.compiler/com.sun.tools.javac.parser=ALL-UNNAMED
                --add-opens=jdk.compiler/com.sun.tools.javac.processing=ALL-UNNAMED
                --add-opens=jdk.compiler/com.sun.tools.javac.tree=ALL-UNNAMED
                --add-opens=jdk.compiler/com.sun.tools.javac.util=ALL-UNNAMED
                '''
                
            }
            
            steps {
                unstash 'source-code'
                
                dir('check-service') {
                    sh 'mvn clean install -DskipTests'
                }
                
                dir('push-service') {
                    sh 'mvn clean install -DskipTests'
                }
            }
        }

        /* =====================================================
           SONARQUBE ANALYSIS
        ===================================================== */

        stage('SonarQube Analysis') {
            agent { label 'workernode2' }
            steps {
                unstash 'source-code'
                script {
                    def scannerHome = tool 'SonarQubeScanner'
                    withSonarQubeEnv('sonarqube') {
                        sh """
                      mvn clean install -DskipTests'
                        -Dsonar.projectKey=microservices \
                        -Dsonar.projectName=microservices \
                        
                        """
                    }
                }
            }
        }

        /* =====================================================
           QUALITY GATE
        ===================================================== */

        stage('Quality Gate') {
            agent { label 'workernode2' }
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        /* =====================================================
           OWASP DEPENDENCY CHECK
        ===================================================== */

        stage('OWASP Dependency Check') {
            agent { label 'workernode2'}
            steps {

                dependencyCheck additionalArguments: '''
                    --scan .
                    --format ALL
                ''',
                odcInstallation: 'OWASP-DC'

                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        /* =====================================================
           DOCKER BUILD
        ===================================================== */

        stage('Docker Build') {
            agent { label 'workernode3' }
            steps {
                unstash 'source-code'
                echo "Build a image for check-service"
                sh """
                docker build -t ${DOCKERHUB_USER}/${IMAGE1}:${BUILD_NUMBER} .
                docker tag ${DOCKERHUB_USER}/${IMAGE1}:${BUILD_NUMBER} ${DOCKERHUB_USER}/${IMAGE1}:latest
                """

                echo "Build a image for push-service"
                sh """
                docker build -t ${DOCKERHUB_USER}/${IMAGE2}:${BUILD_NUMBER} .
                docker tag ${DOCKERHUB_USER}/${IMAGE2}:${BUILD_NUMBER} ${DOCKERHUB_USER}/${IMAGE2}:latest
                """


            }
        }

           /* =====================================================
           TRIVY IMAGE SCAN
        ===================================================== */

        stage('Trivy Scan') {
            agent { label 'workernode3' }
            steps {
                sh """
                trivy image --exit-code 0 --severity HIGH,CRITICAL ${DOCKERHUB_USER}/${IMAGE1}:${BUILD_NUMBER}
                trivy image --exit-code 0 --severity HIGH,CRITICAL ${DOCKERHUB_USER}/${IMAGE2}:${BUILD_NUMBER}
                """
            }
        }

         /* =====================================================
           DOCKER PUSH
        ===================================================== */

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
                docker push ${DOCKERHUB_USER}/${IMAGE1}:${BUILD_NUMBER}
                docker push ${DOCKERHUB_USER}/${IMAGE1}:latest
                """

                 sh """
                docker push ${DOCKERHUB_USER}/${IMAGE2}:${BUILD_NUMBER}
                docker push ${DOCKERHUB_USER}/${IMAGE2}:latest
                """
            }
        }
    }

    post {
        success {
            echo "✅ microservices CI Pipeline SUCCESS"
        }
        failure {
            echo "❌ microservices CI Pipeline FAILED"
        }
    }
}
