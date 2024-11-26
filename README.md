
# Docker Compose Monitoring and Logging Stack

This project provides a monitoring and logging stack based on Docker Compose. It includes 
VictoriaMetrics, VMAgent, Loki, Grafana, Node Exporter, and Promtail for capturing, storing, 
and visualizing metrics and logs.

## Services

### 1. VictoriaMetrics
- **Role**: Time-series database
- **Image**: `victoriametrics/victoria-metrics:latest`
- **Ports**: `8428:8428`
- **Data Volume**: `victoria-metrics-data:/storage`
- **Additional Parameters**:
  - `--storageDataPath=/storage`
  - `--httpListenAddr=:8428`
  - `--retentionPeriod=1` (Data retention in months)

### 2. VMAgent
- **Role**: Data scraping, aggregation, and forwarding to VictoriaMetrics
- **Image**: `victoriametrics/vmagent:latest`
- **Ports**: `8429:8429`
- **Data Volume**: `vmagent-data:/vmagent-data`
- **Configuration File**: `./prometheus.yml` mounted to `/etc/prometheus/prometheus.yml`
- **Additional Parameters**:
  - `--promscrape.config=/etc/prometheus/prometheus.yml`
  - `--promscrape.config.strictParse=false`
  - `--remoteWrite.url=http://victoriametrics:8428/api/v1/write`
  - `--remoteWrite.tmpDataPath=/vmagent-data`

### 3. Loki
- **Role**: Log aggregation
- **Image**: `grafana/loki:latest`
- **Ports**: `3100:3100`
- **Data Volume**: `loki-data:/loki`
- **Configuration File**: `./loki-config.yml` mounted to `/etc/loki/local-config.yaml`

### 4. Grafana
- **Role**: Data visualization and dashboard
- **Image**: `grafana/grafana:latest`
- **Ports**: `3000:3000`
- **Environment Variables**:
  - `GF_SECURITY_ADMIN_PASSWORD=admin123`
  - `GF_USERS_ALLOW_SIGN_UP=false`
- **Data Volume**: `grafana_data:/var/lib/grafana`
- **Configuration Directories**:
  - `./grafana/datasource` to `/etc/grafana/provisioning/datasources`
  - `./grafana/dashboard` to `/etc/grafana/provisioning/dashboards`

### 5. Node Exporter
- **Role**: System metrics exporter
- **Image**: `prom/node-exporter:latest`
- **Ports**: `9100:9100`
- **Mounts**:
  - `/proc`
  - `/sys`
  - `/`
- **Additional Parameters**:
  - `--path.procfs=/host/proc`
  - `--path.rootfs=/rootfs`
  - `--path.sysfs=/host/sys`
  - `--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)`

### 6. Promtail
- **Role**: Log collection and forwarding to Loki
- **Image**: `grafana/promtail:latest`
- **Mounts**:
  - `/var/log:/var/log:ro`
  - `./promtail-config.yml:/etc/promtail/config.yml`
  - `/var/lib/docker/containers:/var/lib/docker/containers:ro`
- **Additional Parameters**:
  - `-config.file=/etc/promtail/config.yml`

## Volumes
- `victoria-metrics-data`
- `vmagent-data`
- `grafana_data`
- `loki-data`

## Configuration Files

### `prometheus.yml`
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'victoriametrics'
    static_configs:
      - targets: [ 'victoriametrics:8428' ]

  - job_name: 'vmagent'
    static_configs:
      - targets: [ 'vmagent:8429' ]

  - job_name: 'node'
    static_configs:
      - targets: [ 'node-exporter:9100' ]

### `loki-config.yml`
auth_enabled: false

server:
  http_listen_port: 3100

common:
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: inmemory
  replication_factor: 1
  path_prefix: /tmp/loki

schema_config:
  configs:
    - from: 2020-05-15
      store: tsdb
      object_store: filesystem
      schema: v13
      index:
        prefix: index_
        period: 24h

storage_config:
  filesystem:
    directory: /tmp/loki/chunks

### `promtail-config.yml`
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: system
    static_configs:
      - targets:
          - localhost
        labels:
          job: varlogs
          __path__: /var/log/*log

  - job_name: docker
    static_configs:
      - targets:
          - localhost
        labels:
          job: docker
          __path__: /var/lib/docker/containers/*/*-json.log
    pipeline_stages:
      - json:
          expressions:
            output: log
            stream: stream
            attrs: attrs
      - labels:
          stream
      - output:
          source: output

### `datasources.yml`
apiVersion: 1

datasources:
  - name: VictoriaMetrics
    type: prometheus
    access: proxy
    url: http://victoriametrics:8428
    jsonData:
      timeInterval: "15s"
      queryTimeout: "60s"
      httpMethod: "POST"
    editable: true
    isDefault: true
  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    jsonData:
      maxLines: 1000
    editable: true

### `dashboards.yml`
apiVersion: 1

providers:
  - name: Default
    folder: ''
    type: file
    disableDeletion: true
    editable: true
    options:
      path: /etc/grafana/provisioning/dashboards

## How to Run

1. **Clone the Repository**.

2. **Navigate to the Project Directory**:
   ```bash
   cd path/to/project
   ```

3. **Start the Docker Compose**:
   ```bash
   docker-compose up -d
   ```

4. **Access Grafana** at [http://localhost:3000](http://localhost:3000) with the default credentials (username: `admin`, password: `admin123`).

5. **Verify Services**:
   - VictoriaMetrics at [http://localhost:8428](http://localhost:8428)
   - Grafana at [http://localhost:3000](http://localhost:3000)
   - Loki at [http://localhost:3100](http://localhost:3100)
   - Node Exporter at [http://localhost:9100](http://localhost:9100)

## Notes

- Adjust the configurations in the YAML files as per your requirements before starting the services.
- Ensure Docker and Docker Compose are installed on your system prior to running the stack.

Enjoy your monitoring and logging stack!
