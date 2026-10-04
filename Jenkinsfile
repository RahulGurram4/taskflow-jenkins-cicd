pipeline {
    agent any

    environment {
        DOCKERHUB_USERNAME = 'gurramrahul'
        IMAGE_TAG = "1.0.${BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Backend Test') {
            steps {
                dir('backend') {
                    sh 'npm ci'
                    sh 'npm test'
                }
            }
        }

        stage('Frontend Build') {
            steps {
                dir('frontend') {
                    sh 'npm ci'
                    sh 'npm run build'
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                sh '''
                    docker build \
                        -t ${DOCKERHUB_USERNAME}/taskflow-backend:${IMAGE_TAG} \
                        ./backend

                    docker build \
                        --build-arg VITE_API_BASE_URL=/api \
                        -t ${DOCKERHUB_USERNAME}/taskflow-frontend:${IMAGE_TAG} \
                        ./frontend
                '''
            }
        }

        stage('Trivy Scan') {
            steps {
                sh '''
                    mkdir -p trivy-reports

                    trivy image \
                        --severity HIGH,CRITICAL \
                        --format table \
                        --output trivy-reports/backend-${IMAGE_TAG}.txt \
                        ${DOCKERHUB_USERNAME}/taskflow-backend:${IMAGE_TAG}

                    trivy image \
                        --severity HIGH,CRITICAL \
                        --format table \
                        --output trivy-reports/frontend-${IMAGE_TAG}.txt \
                        ${DOCKERHUB_USERNAME}/taskflow-frontend:${IMAGE_TAG}
                '''
            }
        }

        stage('Push Docker Images') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login \
                            -u "$DOCKER_USER" \
                            --password-stdin

                        docker push \
                            ${DOCKERHUB_USERNAME}/taskflow-backend:${IMAGE_TAG}

                        docker push \
                            ${DOCKERHUB_USERNAME}/taskflow-frontend:${IMAGE_TAG}

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                sshagent(credentials: ['taskflow-ec2-ssh']) {
                    sh """
                        ssh -o StrictHostKeyChecking=no \
                            ec2-user@ec2-34-224-68-12.compute-1.amazonaws.com '
                                cd /home/ec2-user/taskflow-jenkins-cicd
                                sed -i "s/^IMAGE_TAG=.*/IMAGE_TAG=${IMAGE_TAG}/" .env
                                docker-compose pull
                                docker-compose up -d
                                docker-compose ps
                            '
                    """
                }
            }
        }

        stage('Docker Image Cleanup') {
            steps {
                sh 'docker image prune -f'
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'trivy-reports/*.txt',
                             allowEmptyArchive: true
        }

        success {
            echo 'TaskFlow CI/CD pipeline completed successfully.'
        }

        failure {
            echo 'TaskFlow CI/CD pipeline failed.'
        }
    }
}