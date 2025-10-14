pipeline {
    agent any

    // Parameters used only when manually triggering CD pipeline
    parameters {
        choice(name: 'TARGET_ENV', choices: ['main', 'dev'], description: 'Target environment for manual deploy')
        string(name: 'IMAGE_TAG', defaultValue: 'v1.0', description: 'Docker image tag for manual deploy')
    }

    tools {
        nodejs 'node'
    }

    environment {
        IMAGE_MAIN = 'nodemain'
        IMAGE_DEV  = 'nodedev'
    }

    stages {
        stage('Determine Branch/Environment') {
            steps {
                script {
                    // Use env.BRANCH_NAME if in multibranch pipeline, otherwise use manual TARGET_ENV parameter
                    BRANCH_TO_USE = env.BRANCH_NAME ?: params.TARGET_ENV
                    IMAGE_TAG_TO_USE = params.IMAGE_TAG ?: 'v1.0'

                    echo "Deploying branch/environment: ${BRANCH_TO_USE}"
                    echo "Using Docker image tag: ${IMAGE_TAG_TO_USE}"
                }
            }
        }

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh 'chmod +x scripts/build.sh'
                sh './scripts/build.sh'
            }
        }

        stage('Test') {
            steps {
                sh 'chmod +x scripts/test.sh'
                sh './scripts/test.sh'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    def imageName = (BRANCH_TO_USE == 'main') ? "${IMAGE_MAIN}:${IMAGE_TAG_TO_USE}" : "${IMAGE_DEV}:${IMAGE_TAG_TO_USE}"
                    sh "docker build -t ${imageName} ."
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    def port = (BRANCH_TO_USE == 'main') ? 3000 : 3001
                    def imageName = (BRANCH_TO_USE == 'main') ? "${IMAGE_MAIN}:${IMAGE_TAG_TO_USE}" : "${IMAGE_DEV}:${IMAGE_TAG_TO_USE}"
                    def container = (BRANCH_TO_USE == 'main') ? 'main-container' : 'dev-container'

                    // Stop & remove existing container
                    sh """
                        docker stop ${container} || true
                        docker rm ${container} || true
                    """

                    // Start new container
                    sh """
                        docker run -d --name ${container} --expose ${port} -p ${port}:3000 ${imageName}
                    """

                    echo "App deployed on http://localhost:${port}"
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline completed for branch/environment: ${BRANCH_TO_USE}"
        }
    }
}
