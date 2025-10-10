pipeline {
    agent any

    tools {
        nodejs 'node18'
    }

    environment {
        IMAGE_MAIN = 'nodemain:v1.0'
        IMAGE_DEV = 'nodedev:v1.0'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Install Dependencies') {
            steps {
                sh 'npm install'
            }
        }

        stage('Test') {
            steps {
                sh 'npm test'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    if (env.BRANCH_NAME == 'main') {
                        sh 'docker build -t nodemain:v1.0 .'
                    } else if (env.BRANCH_NAME == 'dev') {
                        sh 'docker build -t nodedev:v1.0 .'
                    }
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    def port = (env.BRANCH_NAME == 'main') ? 3000 : 3001
                    def image = (env.BRANCH_NAME == 'main') ? env.IMAGE_MAIN : env.IMAGE_DEV
                    def container = (env.BRANCH_NAME == 'main') ? 'main-container' : 'dev-container'

                    // Stop & remove existing container
                    sh """
                        docker stop ${container} || true
                        docker rm ${container} || true
                    """

                    // Start new container
                    sh """
                        docker run -d --name ${container} --expose ${port} -p ${port}:3000 ${image}
                    """

                    echo "App deployed on http://localhost:${port}"
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline completed for branch: ${env.BRANCH_NAME}"
        }
    }
}
