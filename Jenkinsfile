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

                    def envData = pipelineUtils.determineBranchEnv(
                        env.BRANCH_NAME,
                        params.TARGET_ENV,
                        params.IMAGE_TAG
                    )

                    env.BRANCH_TO_USE    = envData.branchToUse
                    env.IMAGE_TAG_TO_USE = envData.imageTagToUse
                    env.DOCKER_TAG_FINAL = envData.dockerTagFinal
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

        stage('Dockerfile Lint') {
            steps {
                script {
                    dockerOps.lintDockerfile()
                }
            }
        }

        stage('Build Image') {
            steps {
                script {
                    def hubTag = dockerOps.buildImage(
                        DOCKER_REPO,
                        IMAGE_MAIN,
                        IMAGE_DEV,
                        env.BRANCH_TO_USE,
                        env.IMAGE_TAG_TO_USE,
                        env.DOCKER_TAG_FINAL
                    )
                    env.IMAGE_NAME_HUB = hubTag
                }
            }
        }

        stage('Scan Image') {
            steps {
                script {
                    dockerOps.scanImage(env.IMAGE_NAME_HUB)
                }
            }
        }

        stage('Push Image') {
            steps {
                script {
                    dockerOps.pushImage(DOCKER_CREDENTIALS, env.IMAGE_NAME_HUB)
                }
            }
        }

        stage('Trigger Deploy') {
            steps {
                script {
                    dockerOps.triggerDeploy(env.BRANCH_TO_USE, env.IMAGE_TAG_TO_USE)
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
