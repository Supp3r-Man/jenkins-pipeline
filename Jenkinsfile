pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 15, unit: 'MINUTES')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    environment {
        IMAGE = 'ghcr.io/supp3r-man/jenkins-pipeline'
    }

    stages {
        stage('Prepare') {
            steps {
                script {
                    env.IMAGE_TAG = sh(returnStdout: true, script: 'git rev-parse --short=7 HEAD').trim()
                }
                echo "Building ${env.IMAGE}:${env.IMAGE_TAG}"
            }
        }

        stage('Lint & test') {
            steps {
                // The Dockerfile's `test` stage runs ruff and pytest.
                sh 'docker build --target test .'
            }
        }

        stage('Build image') {
            steps {
                sh 'docker build --build-arg APP_VERSION=$IMAGE_TAG --tag $IMAGE:$IMAGE_TAG .'
            }
        }

        stage('Smoke test') {
            steps {
                // Probe from inside the container, so this works even when Jenkins itself runs in Docker.
                sh '''
                    docker run -d --name smoke-$BUILD_NUMBER $IMAGE:$IMAGE_TAG
                    for i in $(seq 1 15); do
                        if docker exec smoke-$BUILD_NUMBER python -c \
                            "import urllib.request; urllib.request.urlopen('http://localhost:8000/health')"; then
                            exit 0
                        fi
                        sleep 1
                    done
                    docker logs smoke-$BUILD_NUMBER
                    exit 1
                '''
            }
        }

        stage('Push image') {
            // Only publish from main; pull-request and feature builds stop after the smoke test.
            when {
                expression { env.BRANCH_NAME == 'main' || env.GIT_BRANCH == 'origin/main' }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'ghcr-credentials',
                                                  usernameVariable: 'REGISTRY_USER',
                                                  passwordVariable: 'REGISTRY_TOKEN')]) {
                    // Single-quoted so the secret is read by the shell, never interpolated by Groovy.
                    sh '''
                        echo "$REGISTRY_TOKEN" | docker login ghcr.io -u "$REGISTRY_USER" --password-stdin
                        docker push $IMAGE:$IMAGE_TAG
                        docker tag $IMAGE:$IMAGE_TAG $IMAGE:latest
                        docker push $IMAGE:latest
                    '''
                }
            }
        }
    }

    post {
        always {
            sh 'docker rm -f smoke-$BUILD_NUMBER || true'
            sh 'docker logout ghcr.io || true'
        }
        success {
            echo "Pipeline succeeded: ${env.IMAGE}:${env.IMAGE_TAG}"
        }
        failure {
            echo 'Pipeline failed - check the stage logs above.'
        }
    }
}
