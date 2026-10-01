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
                    docker build -t ${DOCKERHUB_USERNAME}/taskflow-backend:${IMAGE_TAG} ./backend
                    docker build --build-arg VITE_API_BASE_URL=/api -t ${DOCKERHUB_USERNAME}/taskflow-frontend:${IMAGE_TAG} ./frontend
                '''
            }
        }

        stage('Trivy Scan') {
            steps {
                sh '''
                    mkdir -p trivy-reports

                    trivy image --format table --output trivy-reports/backend-${IMAGE_TAG}.txt ${DOCKERHUB_USERNAME}/taskflow-backend:${IMAGE_TAG}

                    trivy image --format table --output trivy-reports/frontend-${IMAGE_TAG}.txt ${DOCKERHUB_USERNAME}/taskflow-frontend:${IMAGE_TAG}
                '''
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: 'trivy-reports/*.txt',
                             allowEmptyArchive: true
        }

        success {
            echo "TaskFlow CI pipeline completed successfully."
        }

        failure {
            echo "TaskFlow CI pipeline failed."
        }
    }
}