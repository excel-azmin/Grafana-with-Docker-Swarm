# Grafana-with-Docker-Swarm

# Pre-requisite
  * Docker swarm
  * Traefik
  * Portainer

# Grafana Installation with alertmanager, unsee, prometheus

* Create deploy all appps togather
  
```
nano swarmprom.yml
```

```
version: "3.3"

networks:
  net:
    driver: overlay
    attachable: true
  traefik-public:
    external: true

volumes:
    prometheus: {}
    grafana: {}
    alertmanager: {}

configs:
  dockerd_config:
    file: ./dockerd-exporter/Caddyfile
  node_rules:
    file: ./prometheus/rules/swarm_node.rules.yml
  task_rules:
    file: ./prometheus/rules/swarm_task.rules.yml

services:
  dockerd-exporter:
    image: stefanprodan/caddy
    networks:
      - net
    environment:
      - DOCKER_GWBRIDGE_IP=172.18.0.1
    configs:
      - source: dockerd_config
        target: /etc/caddy/Caddyfile
    deploy:
      mode: global
      resources:
        limits:
          memory: 128M
        reservations:
          memory: 64M

  cadvisor:
    image: gcr.io/cadvisor/cadvisor
    networks:
      - net
    command: -logtostderr -docker_only
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /:/rootfs:ro
      - /var/run:/var/run
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    deploy:
      mode: global
      resources:
        limits:
          memory: 128M
        reservations:
          memory: 64M

  grafana:
    image: stefanprodan/swarmprom-grafana:5.3.4
    networks:
      - default
      - net
      - traefik-public
    environment:
      - GF_SECURITY_ADMIN_USER=${ADMIN_USER:-admin}
      - GF_SECURITY_ADMIN_PASSWORD=${ADMIN_PASSWORD:-admin}
      - GF_USERS_ALLOW_SIGN_UP=false
      - GF_SERVER_ROOT_URL=${GF_SERVER_ROOT_URL}
      - GF_SMTP_ENABLED=${GF_SMTP_ENABLED}
      - GF_SMTP_FROM_ADDRESS=${GF_SMTP_FROM_ADDRESS}
      - GF_SMTP_FROM_NAME=${GF_SMTP_FROM_NAME}
      - GF_SMTP_HOST=${GF_SMTP_HOST}
      - GF_SMTP_USER=${GF_SMTP_USER}
      - GF_SMTP_PASSWORD=${GF_SMTP_PASSWORD}
    volumes:
      - grafana:/var/lib/grafana
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints:
          - node.role == manager
      resources:
        limits:
          memory: 128M
        reservations:
          memory: 64M
      labels:
        - "traefik.enable=true"
        - "traefik.docker.network=traefik-public"
        - "traefik.constraint-label=traefik-public"
        - "traefik.http.routers.swarmprom-grafana-http.rule=Host(`grafana.${DOMAIN?Variable not set}`)"
        - "traefik.http.routers.swarmprom-grafana-http.entrypoints=http"
        - "traefik.http.routers.swarmprom-grafana-http.middlewares=https-redirect"
        - "traefik.http.routers.swarmprom-grafana-https.rule=Host(`grafana.${DOMAIN?Variable not set}`)"
        - "traefik.http.routers.swarmprom-grafana-https.entrypoints=https"
        - "traefik.http.routers.swarmprom-grafana-https.tls=true"
        - "traefik.http.routers.swarmprom-grafana-https.tls.certresolver=le"
        - "traefik.http.services.swarmprom-grafana.loadbalancer.server.port=3000"

  alertmanager:
    image: stefanprodan/swarmprom-alertmanager:v0.14.0
    networks:
      - default
      - net
      - traefik-public
    environment:
      - SLACK_URL=${SLACK_URL:-https://hooks.slack.com/services/TOKEN}
      - SLACK_CHANNEL=${SLACK_CHANNEL:-general}
      - SLACK_USER=${SLACK_USER:-alertmanager}
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
    volumes:
      - alertmanager:/alertmanager
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints:
          - node.role == manager
      resources:
        limits:
          memory: 128M
        reservations:
          memory: 64M
      labels:
        - "traefik.enable=true"
        - "traefik.docker.network=traefik-public"
        - "traefik.constraint-label=traefik-public"
        - "traefik.http.routers.swarmprom-alertmanager-http.rule=Host(`alertmanager.${DOMAIN?Variable not set}`)"
        - "traefik.http.routers.swarmprom-alertmanager-http.entrypoints=http"
        - "traefik.http.routers.swarmprom-alertmanager-http.middlewares=https-redirect"
        - "traefik.http.routers.swarmprom-alertmanager-https.rule=Host(`alertmanager.${DOMAIN?Variable not set}`)"
        - "traefik.http.routers.swarmprom-alertmanager-https.entrypoints=https"
        - "traefik.http.routers.swarmprom-alertmanager-https.tls=true"
        - "traefik.http.routers.swarmprom-alertmanager-https.tls.certresolver=le"
        - "traefik.http.services.swarmprom-alertmanager.loadbalancer.server.port=9093"
        - "traefik.http.middlewares.swarmprom-alertmanager-auth.basicauth.users=${ADMIN_USER?Variable not set}:${HASHED_PASSWORD?Variable not set}"
        - "traefik.http.routers.swarmprom-alertmanager-https.middlewares=swarmprom-alertmanager-auth"

  unsee:
    image: cloudflare/unsee:v0.8.0
    networks:
      - default
      - net
      - traefik-public
    environment:
      - "ALERTMANAGER_URIS=default:http://alertmanager:9093"
    deploy:
      mode: replicated
      replicas: 1
      labels:
        - "traefik.enable=true"
        - "traefik.docker.network=traefik-public"
        - "traefik.constraint-label=traefik-public"
        - "traefik.http.routers.swarmprom-unsee-http.rule=Host(`unsee.${DOMAIN?Variable not set}`)"
        - "traefik.http.routers.swarmprom-unsee-http.entrypoints=http"
        - "traefik.http.routers.swarmprom-unsee-http.middlewares=https-redirect"
        - "traefik.http.routers.swarmprom-unsee-https.rule=Host(`unsee.${DOMAIN?Variable not set}`)"
        - "traefik.http.routers.swarmprom-unsee-https.entrypoints=https"
        - "traefik.http.routers.swarmprom-unsee-https.tls=true"
        - "traefik.http.routers.swarmprom-unsee-https.tls.certresolver=le"
        - "traefik.http.services.swarmprom-unsee.loadbalancer.server.port=8080"
        - "traefik.http.middlewares.swarmprom-unsee-auth.basicauth.users=${ADMIN_USER?Variable not set}:${HASHED_PASSWORD?Variable not set}"
        - "traefik.http.routers.swarmprom-unsee-https.middlewares=swarmprom-unsee-auth"

  node-exporter:
    image: stefanprodan/swarmprom-node-exporter:v0.16.0
    networks:
      - net
    environment:
      - NODE_ID={{.Node.ID}}
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
      - /etc/hostname:/etc/nodename
    command:
      - '--path.sysfs=/host/sys'
      - '--path.procfs=/host/proc'
      - '--collector.textfile.directory=/etc/node-exporter/'
      - '--collector.filesystem.ignored-mount-points=^/(sys|proc|dev|host|etc)($$|/)'
      - '--no-collector.ipvs'
    deploy:
      mode: global
      resources:
        limits:
          memory: 128M
        reservations:
          memory: 64M

  prometheus:
    image: stefanprodan/swarmprom-prometheus:v2.5.0
    networks:
      - default
      - net
      - traefik-public
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention=${PROMETHEUS_RETENTION:-24h}'
    volumes:
      - prometheus:/prometheus
    configs:
      - source: node_rules
        target: /etc/prometheus/swarm_node.rules.yml
      - source: task_rules
        target: /etc/prometheus/swarm_task.rules.yml
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints:
          - node.role == manager
      resources:
        limits:
          memory: 2048M
        reservations:
          memory: 128M
      labels:
        - "traefik.enable=true"
        - "traefik.docker.network=traefik-public"
        - "traefik.constraint-label=traefik-public"
        - "traefik.http.routers.swarmprom-prometheus-http.rule=Host(`prometheus.${DOMAIN?Variable not set}`)"
        - "traefik.http.routers.swarmprom-prometheus-http.entrypoints=http"
        - "traefik.http.routers.swarmprom-prometheus-http.middlewares=https-redirect"
        - "traefik.http.routers.swarmprom-prometheus-https.rule=Host(`prometheus.${DOMAIN?Variable not set}`)"
        - "traefik.http.routers.swarmprom-prometheus-https.entrypoints=https"
        - "traefik.http.routers.swarmprom-prometheus-https.tls=true"
        - "traefik.http.routers.swarmprom-prometheus-https.tls.certresolver=le"
        - "traefik.http.services.swarmprom-prometheus.loadbalancer.server.port=9090"
        - "traefik.http.middlewares.swarmprom-prometheus-auth.basicauth.users=${ADMIN_USER?Variable not set}:${HASHED_PASSWORD?Variable not set}"
        - "traefik.http.routers.swarmprom-prometheus-https.middlewares=swarmprom-prometheus-auth"
```

* Make sure that the following sub-domains point to your Docker Swarm cluster IPs:

```
grafana.example.com
alertmanager.example.com
unsee.example.com
prometheus.example.com
```
* Login to your server and run the command one by one

```
git clone https://github.com/stefanprodan/swarmprom.git

cd swarmprom
export ADMIN_USER=admin
export ADMIN_PASSWORD=gZyU10aUl3AT
export HASHED_PASSWORD=$(openssl passwd -apr1 $ADMIN_PASSWORD)
export HASHED_PASSWORD=$(openssl passwd -apr1)
echo $HASHED_PASSWORD
export DOMAIN=example.com

# Setup you mail server

export GF_SERVER_ROOT_URL=https://example.arcapps.org 
export GF_SMTP_ENABLED=true 
export GF_SMTP_FROM_ADDRESS=example@gmail.com 
export GF_SMTP_FROM_NAME=Grafana 
export GF_SMTP_HOST=smtp.gmail.com:587 
export GF_SMTP_USER=example@gmail.com 
export GF_SMTP_PASSWORD='your app password'
```

* If you are using Slack and want to integrate it, set the following environment variables:

```
export SLACK_URL=https://hooks.slack.com/services/TOKEN 
export SLACK_CHANNEL=devops-alerts 
export SLACK_USER=alertmanager
```

* Deploy the stack

```
docker stack deploy -c swarmprom.yml swarmprom
```
* Set your notification mail
![image](https://github.com/user-attachments/assets/107463d3-50b9-4586-aeda-b0494e9e84e5)

* Set Alert -> Edit Chart -> 
![image](https://github.com/user-attachments/assets/bd0bac00-e65c-4618-bf1c-4dba3c0e1ad2)

* Alert -> set your required condition
![image](https://github.com/user-attachments/assets/b5bb2bc5-1ed9-4cb1-b5c1-114304ff9747)


# Thank you so much don't forget to give me a start 
  
