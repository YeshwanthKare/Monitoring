# Observability

-> In EC2 Instance update the machine

```
sudo apt-get update
```

-> Now Install docker, docker-compose

```
sudo apt-get install docker.io docker-compose-v2 -y
```

-> Check the docker user permissions

```
docker ps
```

-> Now we will add the current user to the docker group and refreshing the docker group

```
sudo usermod -aG docker $USER && newgrp docker
```

- Now the current User have the permission to access the docker

-> Check the docker compose version

```
docker compose version
```

-> Now we will make a directory Observability

```
mkdir observability

cd observability
```

-> Clone any app in the directory

```
git clone <git-repo>

cd <git-repo>
```

-> Checkout a dev branch

```
git checkout dev
```

-> Now build the image of the notes-app

```
docker build -t notes-app .
```

-> Now run the docker container with the notes-app image

```
docker run -d -p 8000:8000 notes-app
```

-> Access the app on the browser with the Public-IP of the virtual machine (EC2 Instance) along with the port 8000

```
http://<public-ip-VM>:8000
```

### Creating a docker-compose file

-> Create a docker-compose file

```
vi docker-compose.yaml

version: "3.8"

networks:
  monitoring:
    driver: bridge

services:

  notes-app:
    build:
      context: django-notes-app/.
    container_name: notes-app
    ports:
      - "8000:8000"

    networks:
      - monitoring

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    networks:
      - monitoring
```

-> Here we created network "Monitoring" because of the fact that the network should be the same and should be bridge for communication

-> Creating Volumes for prometheus, for that we need to create a prometheus configuration file

- Prometheus configuration file - (prometheus.yaml)

```
global:
  scrape_interval: 1m

scrape_configs:
  - job_name: 'prometheus'
    scrape_interval: 1m
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'docker'
    scrape_interval: 1m
    static_configs:
      - targets: ['localhost:8080']
```

-> Now we can add the volume inside the docker-compose.yaml

```
version: "3.8"

networks:
  monitoring:
    driver: bridge

services:

  notes-app:
    build:
      context: django-notes-app/.
    container_name: notes-app
    ports:
      - "8000:8000"

    networks:
      - monitoring

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    networks:
      - monitoring

    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--web.enable-lifecycle'

    ports:
      - "9090:9090"
```

-> Now we can access the prometheus and notes app by

```
docker compose up -d

&&

# in the browser

http://<public-ip-VM>:<port>
```

#### Metric will be stored in following link

```
http://<public-ip-VM>:<port>/metrics
```

- if we go to Target Health, it will say what are the endpoints available
- if the localhost is resembled in Prometheus target Helath, then it checks for the "localhost" within the container of Prometheus
- But if the network is same for other container, then the other container can be accessed by its name

#### cadvisor

- The cadvisor takes all the contents of the container and gives to the Prometheus

```
cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    ports:
      - "8080:8080"
    volumes:
    - /:/rootfs:ro
    - /var/run:/var/run:rw
    - /sys:/sys:ro
    - /var/lib/docker/:/var/lib/docker:ro
    depends_on:
    - redis
    networks:
      - monitoring

  redis:
    image: redis:latest
    container_name: redis
    ports:
    - 6379:6379
    networks:
      - monitoring
```

#### Nodeexporter

- The Node exporter will export all the system processes, like the system logs and the system metrics to the prometheus

```
node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    ports:
      - "9100:9100"
    networks:
      - monitoring
```

#### The full Docker compose file

```
version: "3.8"

networks:
  monitoring:
    driver: bridge

volumes:
  prometheus_data:
  grafana_data:


services:

  notes-app:
    build:
      context: django-notes-app/.
    container_name: notes-app
    ports:
      - "8000:8000"

    networks:
      - monitoring

  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    restart: unless-stopped
    networks:
      - monitoring

    volumes:
      - ./prometheus.yaml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--web.enable-lifecycle'

    ports:
      - "9090:9090"

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    ports:
      - "8080:8080"
    volumes:
    - /:/rootfs:ro
    - /var/run:/var/run:rw
    - /sys:/sys:ro
    - /var/lib/docker/:/var/lib/docker:ro
    depends_on:
    - redis
    networks:
      - monitoring

  redis:
    image: redis:latest
    container_name: redis
    ports:
    - 6379:6379
    networks:
      - monitoring

  node-exporter:
    image: prom/node-exporter:latest
    container_name: node-exporter
    restart: unless-stopped
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    ports:
      - "9100:9100"
    networks:
      - monitoring

  grafana:
    image: grafana/grafana-enterprise:latest
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    networks:
      - monitoring
    restart: unless-stopped
    depends_on:
      - prometheus
```
