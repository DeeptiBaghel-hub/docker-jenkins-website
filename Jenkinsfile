
pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        disableConcurrentBuilds()
        timestamps()
    }

    environment {
        IMAGE_NAME = 'my-website'
        CONTAINER_NAME = 'my-website'
        HOST_PORT = '8080'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Validate Website') {
            steps {
                sh '''
                    set -eu
                    test -s index.html
                    test -s style.css
                    test -s script.js
                    test -s Dockerfile
                    echo "Website files validated"
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh '''
                    docker build --pull \
                      -t ${IMAGE_NAME}:${BUILD_NUMBER} .
                    docker tag ${IMAGE_NAME}:${BUILD_NUMBER} \
                      ${IMAGE_NAME}:latest
                '''
            }
        }

        stage('Test Docker Image') {
            steps {
                sh '''
                    set -eu
                    docker run -d --rm \
                      --name website-test-${BUILD_NUMBER} \
                      ${IMAGE_NAME}:${BUILD_NUMBER}

                    trap 'docker stop website-test-${BUILD_NUMBER} >/dev/null 2>&1 || true' EXIT

                    for i in $(seq 1 15); do
                      if docker exec website-test-${BUILD_NUMBER} \
                        wget -q -O /dev/null http://127.0.0.1/; then
                        echo "Website HTTP test passed"
                        exit 0
                      fi
                      sleep 2
                    done

                    echo "Website HTTP test failed"
                    exit 1
                '''
            }
        }

        stage('Deploy Website') {
            steps {
                sh '''
                    set -eu

                    if docker ps -a --format '{{.Names}}' |
                       grep -Fxq "${CONTAINER_NAME}"; then
                        docker rm -f "${CONTAINER_NAME}"
                    fi

                    docker run -d \
                      --name "${CONTAINER_NAME}" \
                      --restart unless-stopped \
                      -p "${HOST_PORT}:80" \
                      "${IMAGE_NAME}:${BUILD_NUMBER}"

                    echo "Deployment completed"
                '''
            }
        }
    }

    post {
        success {
            echo 'CI/CD SUCCESS: Website deployed!'
        }
        failure {
            echo 'CI/CD FAILED: Check the Jenkins console output.'
        }
    }
}
