pipeline {
    agent any
    environment {
        IMAGES = 'redis:7-alpine eclipse-mosquitto:2 httpd:2.4 mcr.microsoft.com/dotnet/samples:aspnetapp'
        S1_SERVICES = 'hes-redis mqtt-service hes-portal restapi akka-service push-service'
        S2_SERVICES = 'dlms-service commissioning-service prepayment-service bridge-service modbus-service reader-service timer-service'
    }
    stages {
        stage('Build (pull + save official images)') {
            steps {
                sh '''
                    rm -f prod/docker/*.tar
                    for img in $IMAGES; do
                        docker pull "$img"
                        docker save -o "prod/docker/$(echo "$img" | tr '/:' '__').tar" "$img"
                    done
                    ls -lh prod/docker/*.tar
                '''
            }
        }
        stage('Deploy') {
            steps {
                sshPublisher(publishers: [
                    sshPublisherDesc(configName: 'physical-server-1', verbose: true, transfers: [
                        sshTransfer(sourceFiles: 'prod/docker/**', removePrefix: 'prod/docker', remoteDirectory: '',
                            execTimeout: 600000,
                            execCommand: 'cd /home/ubuntu && for f in *.tar; do docker load -i "$f"; done && docker compose up -d ' + env.S1_SERVICES + ' && docker compose ps'
                    ]),
                    sshPublisherDesc(configName: 'physical-server-2', verbose: true, transfers: [
                        sshTransfer(sourceFiles: 'prod/docker/**', removePrefix: 'prod/docker', remoteDirectory: '',
                            execTimeout: 600000,
                            execCommand: 'cd /home/ubuntu && for f in *.tar; do docker load -i "$f"; done && docker compose up -d ' + env.S2_SERVICES + ' && docker compose ps'
                    ])
                ])
            }
        }
    }
}
