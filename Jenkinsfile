pipeline {
    agent any

    // Parameters for manual CD deployment
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
                    // Declare variables with def
                    def BRANCH_TO_USE = env.BRANCH_NAME ?: params.TARGET_ENV
                    def IMAGE_TAG_TO_USE = params.IMAGE_TAG ?: 'v1.0'

                    // Store them in env so they can be used in other stages
                    env.BRANCH_TO_USE = BRANCH_TO_USE
                    env.IMAGE_TAG_TO_USE = IMAGE_TAG_TO_USE

                    echo "Deploying branch/environment: ${env.BRANCH_TO_USE}"
                    echo "Using Docker image tag: ${env.IMAGE_TAG_TO_USE}"
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
                    def imageName = (env.BRANCH_TO_USE == 'main') ? "${IMAGE_MAIN}:${env.IMAGE_TAG_TO_USE}" : "${IMAGE_DEV}:${env.IMAGE_TAG_TO_USE}"
                    sh "docker build -t ${imageName} ."
                }
            }
        }

        stage('Deploy') {
            steps {
                script {
                    def port = (env.BRANCH_TO_USE == 'main') ? 3000 : 3001
                    def imageToRun = (env.BRANCH_TO_USE == 'main') ? "${IMAGE_MAIN}:${env.IMAGE_TAG_TO_USE}" : "${IMAGE_DEV}:${env.IMAGE_TAG_TO_USE}"
                    def containerName = "${env.BRANCH_TO_USE}-container-${env.IMAGE_TAG_TO_USE}"

                    // Stop & remove old containers matching branch
                    sh """
                        docker ps -a --filter "name=${env.BRANCH_TO_USE}-container" --format "{{.Names}}" | xargs -r docker stop
                        docker ps -a --filter "name=${env.BRANCH_TO_USE}-container" --format "{{.Names}}" | xargs -r docker rm
                    """

                    // Run new container
                    sh "docker run -d --name ${containerName} --expose ${port} -p ${port}:3000 ${imageToRun}"

                    echo "App deployed on http://localhost:${port}"
                }
            }
        }
    }

    post {
        always {
            echo "Pipeline completed for branch/environment: ${env.BRANCH_TO_USE}"
        }
    }
}
