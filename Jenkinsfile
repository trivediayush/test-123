pipeline {
    agent any

    environment {
        DEPLOY_DIR = '/opt/deployment'

        IMAGES = 'redis:7-alpine eclipse-mosquitto:2 httpd:2.4 mcr.microsoft.com/dotnet/samples:aspnetapp'
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Source code checked out from GitHub.'
            }
        }

        stage('Build - Pull and Save Docker Images') {
            steps {
                sh '''
                    set -e

                    cd prod/docker

                    echo "Pulling required Docker images..."

                    for img in $IMAGES; do
                        docker pull "$img"

                        filename=$(echo "$img" | tr '/:' '__').tar

                        echo "Saving $img as $filename"

                        docker save -o "$filename" "$img"
                    done

                    echo "Generated image archives:"
                    ls -lh *.tar
                '''
            }
        }

        stage('Validate Docker Compose') {
            steps {
                sh '''
                    set -e

                    echo "Checking required deployment files..."

                    test -f prod/docker/docker-compose.yml
                    test -f prod/docker/Services/Mqtt/mosquitto.conf
                    test -f prod/docker/Services/Portal/index.html
                    test -f prod/docker/Services/Redis/redis.conf

                    echo "Validating Compose configuration..."

                    docker compose -f prod/docker/docker-compose.yml config -q

                    echo "Validation successful."
                '''
            }
        }

        stage('Deploy on Jenkins Server') {
            steps {
                sh '''
                    set -e

                    echo "Preparing Jenkins deployment directory..."

                    mkdir -p "$DEPLOY_DIR"

                    cp -a prod/docker/. "$DEPLOY_DIR/"

                    cd "$DEPLOY_DIR"

                    echo "Loading Docker images..."

                    for f in *.tar; do
                        echo "Loading $f"
                        docker load -i "$f"
                    done

                    echo "Starting all services on Jenkins server..."

                    docker compose up -d

                    echo "Jenkins server container status:"
                    docker compose ps
                '''
            }
        }

        stage('Deploy on Physical Server 1') {
            steps {
                sshPublisher(
                    publishers: [
                        sshPublisherDesc(
                            configName: 'physical-server-1',
                            verbose: true,
                            transfers: [
                                sshTransfer(
                                    sourceFiles: 'prod/docker/**',
                                    removePrefix: 'prod/docker',
                                    remoteDirectory: '',
                                    execTimeout: 600000,
                                    execCommand: 'set -e; cd /opt/deployment && echo "Loading Docker images..." && for f in *.tar; do echo "Loading $f"; docker load -i "$f"; done && echo "Starting all services..." && docker compose up -d && echo "Container status:" && docker compose ps'
                                )
                            ]
                        )
                    ]
                )
            }
        }

        stage('Deploy on Physical Server 2') {
            steps {
                sshPublisher(
                    publishers: [
                        sshPublisherDesc(
                            configName: 'physical-server-2',
                            verbose: true,
                            transfers: [
                                sshTransfer(
                                    sourceFiles: 'prod/docker/**',
                                    removePrefix: 'prod/docker',
                                    remoteDirectory: '',
                                    execTimeout: 600000,
                                    execCommand: 'set -e; cd /opt/deployment && echo "Loading Docker images..." && for f in *.tar; do echo "Loading $f"; docker load -i "$f"; done && echo "Starting all services..." && docker compose up -d && echo "Container status:" && docker compose ps'
                                )
                            ]
                        )
                    ]
                )
            }
        }

    }

    post {
        success {
            echo 'SUCCESS: All services deployed on all 3 servers.'
        }

        failure {
            echo 'FAILED: Deployment stopped. Check the stage logs above.'
        }

        always {
            echo 'Jenkins pipeline execution completed.'
        }
    }
}
