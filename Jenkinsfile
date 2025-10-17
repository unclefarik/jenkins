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
	DOCKER_REPO = 'unckefarik/node-app'
        DOCKER_CREDENTIALS = 'dockerhub-creds'
    }

    stages {
        stage('Determine Branch/Environment') {
            steps {
                script {
                    // Declare variables with def
                    def BR = env.BRANCH_NAME ?: params.TARGET_ENV
                    def TAG = params.IMAGE_TAG ?: 'v1.0'

                    env.BRANCH_TO_USE = BR
                    env.IMAGE_TAG_TO_USE = TAG

                    // final tag for Docker Hub: "main-v1.0" or "dev-v1.0"
                    env.DOCKER_TAG_FINAL = "${BR}-${TAG}"

                    echo "Deploying branch/environment: ${env.BRANCH_TO_USE}"
                    echo "Using Docker image tag: ${env.IMAGE_TAG_TO_USE}"
                    echo "Docker Hub tag to use: ${env.DOCKER_TAG_FINAL}"
                }
            }
        }

        stage('Checkout') {
            steps {
                checkout scm
            }
        
	// ---------- NEW: Hadolint (Dockerfile linter) ----------
        stage('Lint Dockerfile (Hadolint)') {
            steps {
                // run hadolint via docker image
                sh '''
                    echo "Running hadolint..."
                    docker run --rm -i hadolint/hadolint:latest < Dockerfile || true
                '''
                // remove "|| true" to fail on lint errors
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    // local image name
                    def localImage = (env.BRANCH_TO_USE == 'main') ? "${IMAGE_MAIN}:${env.IMAGE_TAG_TO_USE}" : "${IMAGE_DEV}:${env.IMAGE_TAG_TO_USE}"
                    // docker hub tag (single repo with branch-tag pattern)
                    def hubTag = "${DOCKER_REPO}:${env.DOCKER_TAG_FINAL}"

                    // build and tag both local and docker-hub tag
                    sh "docker build -t ${localImage} -t ${hubTag} ."

                    // export name for next stages
                    env.IMAGE_NAME_LOCAL = localImage
                    env.IMAGE_NAME_HUB = hubTag

                    echo "Built local image: ${env.IMAGE_NAME_LOCAL}"
                    echo "Also tagged for push: ${env.IMAGE_NAME_HUB}"
                }
            }
        }

	// ---------- NEW: Trivy vulnerability scan ----------
        stage('Scan image with Trivy') {
            steps {
                script {
                    // run trivy via docker image
                    sh """
                      echo "Running Trivy scan for ${env.IMAGE_NAME_LOCAL} (HIGH/CRITICAL)..."
                      docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy:latest image --severity HIGH,CRITICAL ${env.IMAGE_NAME_LOCAL} || true
                    """
                }
            }
        }

        // ---------- NEW: Push image to Docker Hub ----------
        stage('Push to Docker Hub') {
            steps {
                script {
                    // login and push using Jenkins credential id
                    docker.withRegistry('https://index.docker.io/v1/', DOCKER_CREDENTIALS) {
                        sh "docker push ${env.IMAGE_NAME_HUB}"
                    }
                    echo "Pushed ${env.IMAGE_NAME_HUB} to Docker Hub"
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
		    // Pull the pushed image from Docker Hub to ensure consistent source
                    sh "docker pull ${DOCKER_REPO}:${env.DOCKER_TAG_FINAL} || true"

                    // Run new container
                    sh "docker run -d --name ${containerName} --expose ${port} -p ${port}:3000 ${DOCKER_REPO}:${env.DOCKER_TAG_FINAL}"

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
