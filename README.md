# HES stack – AWS test guide (official images)

## What the project is
HES = Head-End System for smart meters. Real stack: Redis + MQTT broker (Aedes/Node) + Apache portal
+ ~10 .NET services (REST API, Akka, DLMS, Modbus, Commissioning, Prepayment, Push, Bridge, Reader, Timer).
All use `network_mode: host`, `restart: always`, json-file log rotation. Jenkins ships images as .tar
over SSH (Publish Over SSH) -> server does `docker load` + `docker compose up`.

## Test substitutes (no source available)
| Original | Test image | Port (host network) |
|---|---|---|
| redis:7-alpine | redis:7-alpine (same) | 6379 |
| hes_mqtt (Aedes) | eclipse-mosquitto:2 | 1883 MQTT, 3000 WS |
| hes_portal (Apache) | httpd:2.4 | 80 |
| hes_api | mcr.microsoft.com/dotnet/samples:aspnetapp | 5002 |
| hes_akka | same aspnetapp | 9083 |
| hes_push | same aspnetapp | 4061 |
| dlms/commissioning/prepayment/bridge/modbus/reader/timer | same aspnetapp | 5101-5107 |

Split: server 1 = redis, mqtt, portal, restapi, akka, push. Server 2 = the 7 worker services.

## Step 1 – Security group "physical server" (all 3 instances share it)
Inbound rules:
- 22 TCP   from My IP            (your SSH)
- 22 TCP   from sg "physical server" itself (Jenkins -> servers)
- 8080 TCP from My IP            (Jenkins UI)
- 80, 1883, 3000, 5002, 9083, 4061, 5101-5107 TCP from My IP (testing)
- 6379, 1883 TCP from sg itself  (server 2 -> server 1)

## Step 2 – Docker on all 3 instances
Ubuntu:
    curl -fsSL https://get.docker.com | sudo sh
    sudo usermod -aG docker $USER
Amazon Linux 2023:
    sudo dnf install -y docker && sudo systemctl enable --now docker
    sudo usermod -aG docker ec2-user
    sudo mkdir -p /usr/local/lib/docker/cli-plugins
    sudo curl -SL https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64 -o /usr/local/lib/docker/cli-plugins/docker-compose
    sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose
Log out/in, then check: `docker compose version`.

## Step 3 – Physical servers 1 and 2
    sudo mkdir -p /opt/deployment && sudo chown $USER:$USER /opt/deployment

## Step 4 – Jenkins instance
    sudo usermod -aG docker jenkins && sudo systemctl restart jenkins
Jenkins UI -> Manage Jenkins -> Plugins -> install "Publish Over SSH" (and "Git" if missing).
Manage Jenkins -> System -> Publish over SSH:
- Key: paste the full content of mumbai-key.pem
- Add SSH Server: Name `physical-server-1`, Hostname = PRIVATE IP of physical-server 1,
  Username `ubuntu` (or `ec2-user`), Remote Directory `/opt/deployment` -> Test Configuration -> Success
- Add SSH Server: Name `physical-server-2`, same with private IP of physical server 2
Save.

## Step 5 – Put this bundle in a git repo
Push this folder (Jenkinsfile + prod/docker/...) to a GitHub repo.
Jenkins -> New Item -> Pipeline -> "Pipeline script from SCM" -> Git -> repo URL (+ credentials if private)
-> Script Path `Jenkinsfile` -> Save -> Build Now.

What the pipeline does:
1. Build: docker pull 4 official images, docker save each to prod/docker/*.tar
2. Deploy: Publish Over SSH copies prod/docker/** (tars + compose + configs) to /opt/deployment on
   both servers, then runs docker load + docker compose up -d <that server's services>.

## Step 6 – Verify
On server 1:
    cd /opt/deployment && docker compose ps            # 6 containers Up
    docker exec hes_redis_container redis-cli -a ChangeMe123 --no-auth-warning ping   # PONG
    docker exec hes_redis_container redis-cli -a ChangeMe123 --no-auth-warning flushall # ERR unknown command (disabled)
    curl localhost:80 ; curl -I localhost:5002 ; curl -I localhost:9083 ; curl -I localhost:4061
On server 2:
    cd /opt/deployment && docker compose ps            # 7 containers Up
    for p in 5101 5102 5103 5104 5105 5106 5107; do curl -s -o /dev/null -w "$p %{http_code}\n" localhost:$p; done
Cross-server MQTT (S1 = private IP of server 1):
    server 1: docker exec hes-mqtt-container mosquitto_sub -h localhost -t test -v
    server 2: docker run --rm --network host eclipse-mosquitto:2 mosquitto_pub -h S1 -t test -m hello
    -> server 1 prints "test hello"
Cross-server Redis:
    server 2: docker run --rm redis:7-alpine redis-cli -h S1 -a ChangeMe123 ping   # PONG
Browser: http://13.234.30.154 (portal), http://13.234.30.154:5002, http://3.110.209.172:5101
Restart policy: `sudo reboot` a server -> all its containers come back Up.
Log rotation: `docker inspect -f '{{.HostConfig.LogConfig}}' hes-restapi-container` -> max-size 100m, max-file 5.

## Troubleshooting
- Jenkins "permission denied /var/run/docker.sock" -> jenkins not in docker group / not restarted.
- Publish Over SSH "Auth fail" -> wrong username, or key not RSA PEM.
- Docker Hub "429 Too Many Requests" -> pull from mirror.gcr.io/library/<image> and `docker tag` it.
- Exec timeout on deploy -> raise execTimeout (aspnetapp tar ~ 200 MB load on t3.small).
- Cleanup: `cd /opt/deployment && docker compose down -v && rm -f *.tar`
