
pipeline {
    agent any

    environment {
        IMAGES = 'redis:7-alpine eclipse-mosquitto:2 httpd:2.4 mcr.microsoft.com/dotnet/samples:aspnetapp'

        S1_SERVICES = 'hes-redis mqtt-service hes-portal restapi akka-service push-service'

        S2_SERVICES = 'dlms-service commissioning-service prepayment-service bridge-service modbus-service reader-service timer-service'

        DEPLOY_DIR = '/opt/deployment'
    }

    stages {

        stage('Checkout') {
            steps {
                echo 'Source code checked out by Jenkins Pipeline SCM.'
            }
        }

        stage('Build - Pull and Save Docker Images') {
            steps {
                sh '''
                    set -e

                    echo "Cleaning old image archives..."
                    rm -f prod/docker/*.tar

                    echo "Pulling and saving Docker images..."

                    for img in $IMAGES; do
                        echo "Pulling: $img"
                        docker pull "$img"

                        filename=$(echo "$img" | tr '/:' '__').tar

                        echo "Saving: $filename"
                        docker save -o "prod/docker/$filename" "$img"
                    done

                    echo "Generated image archives:"
                    ls -lh prod/docker/*.tar
                '''
            }
        }

        stage('Validate Deployment Files') {
            steps {
                sh '''
                    set -e

                    test -f prod/docker/docker-compose.yml
                    test -f prod/docker/Services/Mqtt/mosquitto.conf
                    test -f prod/docker/Services/Redis/redis.conf
                    test -f prod/docker/Services/Portal/index.html

                    echo "Deployment files verified."
                '''
            }
        }

        stage('Deploy to Physical Server 1') {
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
                                    execCommand: 'set -e && cd /opt/deployment && for f in *.tar; do echo "Loading $f"; docker load -i "$f"; done && docker compose up -d ' + env.S1_SERVICES + ' && docker compose ps'
                                )
                            ]
                        )
                    ]
                )
            }
        }

        stage('Deploy to Physical Server 2') {
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
                                    execCommand: 'set -e && cd /opt/deployment && for f in *.tar; do echo "Loading $f"; docker load -i "$f"; done && docker compose up -d ' + env.S2_SERVICES + ' && docker compose ps'
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
            echo 'Deployment completed successfully on both servers.'
        }

        failure {
            echo 'Pipeline failed. Check the stage logs above for the exact error.'
        }

        always {
            echo 'Jenkins pipeline execution finished.'
        }
    }
}
