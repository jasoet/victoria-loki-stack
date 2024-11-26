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

Configuration for Prometheus scraping jobs.

### `loki-config.yml`

Configuration for Loki server.

### `promtail-config.yml`

Configuration for Promtail log collection.

### `datasources.yml`

Configuration for Grafana data sources.

### `dashboards.yml`

Configuration for Grafana dashboards.

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
