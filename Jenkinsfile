@Library('cicd-sharedlib') _

pipeline {
    agent {
        docker {
            image 'node:14-alpine'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }

    parameters {
        choice(name: 'TARGET_ENV', choices: ['main', 'dev'], description: 'Target environment for manual deploy')
        string(name: 'IMAGE_TAG', defaultValue: 'v1.0', description: 'Docker image tag for manual deploy')
    }

    environment {
        IMAGE_MAIN = 'nodemain'
        IMAGE_DEV  = 'nodedev'
        DOCKER_REPO = 'unclefarik/node-app'
        DOCKER_CREDENTIALS = 'dockerhub-creds'
    }

    stages {
        stage('Setup') {
            steps {
                script {
                    pipelineUtils.prepareTools()
                    pipelineUtils.determineBranchEnv()
                }
            }
        }

        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Build & Test') {
            steps {
                sh 'chmod +x scripts/build.sh && ./scripts/build.sh'
                sh 'chmod +x scripts/test.sh && ./scripts/test.sh'
            }
        }

        stage('Push to Docker Hub') {
            steps {
                script {
                    dockerOps.buildAndPush(DOCKER_REPO, DOCKER_CREDENTIALS, IMAGE_MAIN, IMAGE_DEV)
                }
            }
        }

        stage('Trigger Deploy Pipeline') {
            steps {
                script {
                    dockerOps.triggerDeploy()
                }
            }
        }
    }

    post {
        always {
            sh 'docker logout https://index.docker.io/v1/ || true'
            echo "Pipeline done for ${env.BRANCH_TO_USE}"
        }
    }
}
