pipeline {
    agent { label 'worker' }

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-creds')
        DOCKER_IMAGE = "mhussain8621/react-docker-app"
        IMAGE_TAG = "jenkins-${BUILD_NUMBER}"
        EC2_HOST = "98.90.194.84"
        SONARQUBE_ENV = "MySonarQube"
        SCANNER_HOME = tool 'SonarScanner'
        NOTIFY_EMAIL = "muhammad.hussain@camp2.tkxel.com"
    }

    stages {

        stage('Checkout Multiple SCM') {
            steps {
                dir('react-docker-app') {
                    git branch: 'main', url: 'https://github.com/hussainsafdar/react-docker-app.git'
                }
                dir('pipeline-config') {
                    git branch: 'main', url: 'https://github.com/hussainsafdar/pipeline-config.git'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                dir('react-docker-app') {
                    withSonarQubeEnv("${SONARQUBE_ENV}") {
                        sh '''
                            ${SCANNER_HOME}/bin/sonar-scanner \
                            -Dsonar.projectKey=react-docker-app \
                            -Dsonar.sources=src
                        '''
                    }
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                dir('react-docker-app') {
                    sh "docker build --platform linux/amd64 -t ${DOCKER_IMAGE}:${IMAGE_TAG} ."
                }
            }
        }

        stage('Push to Docker Hub') {
            steps {
                sh "echo ${DOCKERHUB_CREDENTIALS_PSW} | docker login -u ${DOCKERHUB_CREDENTIALS_USR} --password-stdin"
                sh "docker push ${DOCKER_IMAGE}:${IMAGE_TAG}"
            }
        }

        stage('Deploy to Remote EC2') {
            steps {
                sshagent(credentials: ['ec2-ssh-key']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ec2-user@${EC2_HOST} '
                            sudo docker pull ${DOCKER_IMAGE}:${IMAGE_TAG} &&
                            sudo docker rm -f react-app-live || true &&
                            sudo docker run -d --name react-app-live -p 8082:80 ${DOCKER_IMAGE}:${IMAGE_TAG}
                        '
                    """
                }
            }
        }
    }

    post {
        success {
            echo " SUCCESS: Pipeline completed successfully! Image ${DOCKER_IMAGE}:${IMAGE_TAG} deployed to ${EC2_HOST}:8082"
            emailext (
                subject: "SUCCESS: Pipeline #${BUILD_NUMBER} - ${env.JOB_NAME}",
                body: """
                    <p>Pipeline completed successfully!</p>
                    <p><b>Image:</b> ${DOCKER_IMAGE}:${IMAGE_TAG}</p>
                    <p><b>Deployed to:</b> ${EC2_HOST}:8082</p>
                    <p><b>Console:</b> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                """,
                mimeType: 'text/html',
                to: "${NOTIFY_EMAIL}"
            )
        }
        failure {
            echo " FAILURE: Pipeline failed. Check console logs for build #${BUILD_NUMBER}"
            emailext (
                subject: "FAILURE: Pipeline #${BUILD_NUMBER} - ${env.JOB_NAME}",
                body: """
                    <p>Pipeline failed. Check console logs for build #${BUILD_NUMBER}.</p>
                    <p><b>Console:</b> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a></p>
                """,
                mimeType: 'text/html',
                to: "${NOTIFY_EMAIL}"
            )
        }
    }
}