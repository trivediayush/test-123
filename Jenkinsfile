
pipeline {
    agent any

    environment {
        IMAGES = 'redis:7-alpine eclipse-mosquitto:2 httpd:2.4 mcr.microsoft.com/dotnet/samples:aspnetapp'
        DEPLOY_DIR = '/opt/deployment'
    }

    stages {

        stage('Build - Pull and Save Images') {
            steps {
                sh '''
                    set -e

                    mkdir -p prod/docker
                    rm -f prod/docker/*.tar

                    for img in $IMAGES; do
                        echo "Pulling image: $img"
                        docker pull "$img"

                        filename=$(echo "$img" | tr '/:' '__').tar

                        echo "Saving image: $filename"
                        docker save -o "prod/docker/$filename" "$img"
                    done

                    ls -lh prod/docker/*.tar
                '''
            }
        }

        stage('Validate Deployment Files') {
            steps {
                sh '''
                    set -e

                    test -f prod/docker/docker-compose.yml

                    echo "Docker Compose file found."
                    echo "Deployment files:"
                    find prod/docker -type f
                '''
            }
        }

        stage('Deploy on Jenkins Server') {
            steps {
                sh '''
                    set -e

                    echo "Preparing Jenkins server deployment..."

                    sudo mkdir -p "$DEPLOY_DIR"
                    sudo cp -a prod/docker/. "$DEPLOY_DIR/"
                    sudo chown -R jenkins:jenkins "$DEPLOY_DIR"

                    cd "$DEPLOY_DIR"

                    echo "Loading Docker images..."
                    for f in *.tar; do
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
                                    execCommand: 'set -e && cd /opt/deployment && for f in *.tar; do docker load -i "$f"; done && docker compose up -d && docker compose ps'
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
                                    execCommand: 'set -e && cd /opt/deployment && for f in *.tar; do docker load -i "$f"; done && docker compose up -d && docker compose ps'
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
            echo 'All services deployed successfully on Jenkins and both physical servers.'
        }

        failure {
            echo 'Deployment failed. Check the failed stage in Console Output.'
        }

        always {
            echo 'Pipeline execution completed.'
        }
    }
}
