## 📸 Screenshot

![Grafana-Prometheus](https://github.com/atulkamble/ec2-grafana-prometheus-project/blob/main/Grafana-Prometheus.png)

---

amazon linux | t3.medium | garafana.key 
ssh 

SG: 22, 3000, 9090, 9100

// full installation 

sudo yum install git -y 
git clone https://github.com/atulkamble/ec2-grafana-prometheus-project.git
cd ec2-grafana-prometheus-project
chmod +x script.sh 
sudo su
./script.sh 

http://instance-public-ip:3000 | admin >> admin
http://instance-public-ip:9090
http://instance-public-ip:9100

login to garafana 

add data sources >> select prometheus 

http://localhost:9090 >> save & test 





# ec2 | t3.medium | 20GB SSD | key.pem
```
cd Downloads
chmod 400 key.pem
ssh -i "key.pem" ec2-user@ec2-18-208-192-186.compute-1.amazonaws.com
```

# SG 22, 3000, 9090, 9100

```
sudo yum update -y
sudo yum install git -y
git clone https://github.com/atulkamble/ec2-grafana-prometheus-project.git
cd ec2-grafana-prometheus-project
sudo -i passwd
sudo su 
sudo chmod +x script.sh 
./script.sh
```

# script.sh
```
#!/bin/bash

# Exit on error
set -e

# Ensure root
if [ "$EUID" -ne 0 ]; then
  echo "❌ Please run as root or use sudo"
  exit 1
fi

# Set versions
PROM_VERSION="2.52.0"
NODE_EXPORTER_VERSION="1.8.0"
GRAFANA_RPM="https://dl.grafana.com/enterprise/release/grafana-enterprise-12.0.2-1.x86_64.rpm"

echo "🔄 Updating system packages..."
dnf update -y

############################################
# 1. Grafana Installation
############################################
echo "📦 Installing Grafana Enterprise..."
dnf install -y $GRAFANA_RPM || echo "⚠️ Grafana may already be installed"
systemctl enable --now grafana-server
grafana_version=$(grafana-server -v | head -n 1)
echo "✅ Grafana installed successfully. Version: $grafana_version"

############################################
# 2. Prometheus Installation
############################################

# Add Prometheus user if not exists
if id "prometheus" &>/dev/null; then
    echo "ℹ️ User 'prometheus' already exists."
else
    useradd --no-create-home --shell /sbin/nologin prometheus
fi

# Create necessary directories
mkdir -p /etc/prometheus /var/lib/prometheus
chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus

# Download and extract Prometheus
cd /tmp
wget -q https://github.com/prometheus/prometheus/releases/download/v$PROM_VERSION/prometheus-$PROM_VERSION.linux-amd64.tar.gz
tar -xzf prometheus-$PROM_VERSION.linux-amd64.tar.gz
cd prometheus-$PROM_VERSION.linux-amd64

# Install binaries and configs
install -m 0755 prometheus promtool /usr/local/bin/
cp -r consoles console_libraries /etc/prometheus/
cp prometheus.yml /etc/prometheus/
touch /etc/prometheus/rules.yml
chown -R prometheus:prometheus /etc/prometheus /usr/local/bin/prometheus /usr/local/bin/promtool

# Create systemd service
cat <<EOF > /etc/systemd/system/prometheus.service
[Unit]
Description=Prometheus Monitoring
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \\
  --config.file=/etc/prometheus/prometheus.yml \\
  --storage.tsdb.path=/var/lib/prometheus/ \\
  --web.console.templates=/etc/prometheus/consoles \\
  --web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reexec
systemctl daemon-reload
systemctl enable --now prometheus
echo "✅ Prometheus installed and running."

############################################
# 3. Node Exporter Installation
############################################

# Add Node Exporter user if not exists
if id "node_exporter" &>/dev/null; then
    echo "ℹ️ User 'node_exporter' already exists."
else
    useradd --no-create-home --shell /sbin/nologin node_exporter
fi

cd /tmp
wget -q https://github.com/prometheus/node_exporter/releases/download/v$NODE_EXPORTER_VERSION/node_exporter-$NODE_EXPORTER_VERSION.linux-amd64.tar.gz
tar -xzf node_exporter-$NODE_EXPORTER_VERSION.linux-amd64.tar.gz

# Stop node_exporter if already running
if systemctl is-active --quiet node_exporter; then
    echo "⏹️ Stopping node_exporter before update..."
    systemctl stop node_exporter
fi

# Replace binary
cp -f node_exporter-$NODE_EXPORTER_VERSION.linux-amd64/node_exporter /usr/local/bin/
chown node_exporter:node_exporter /usr/local/bin/node_exporter

# Create Node Exporter systemd service
cat <<EOF > /etc/systemd/system/node_exporter.service
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now node_exporter
echo "✅ Node Exporter installed and running."

############################################
# Summary
############################################
echo -e "\n🎉 Monitoring stack setup complete!"
echo "➡️ Prometheus: http://<your-ec2-ip>:9090"
echo "➡️ Grafana:    http://<your-ec2-ip>:3000 (login: admin / admin)"
echo "➡️ Node Exporter: http://<your-ec2-ip>:9100"
```
Here’s a complete beginner-to-advanced **Prometheus + Grafana** tutorial tailored for EC2 or local Linux environments, ideal for monitoring cloud-native infrastructure or services.

---

# 🚀 Prometheus + Grafana Monitoring Stack Tutorial

## 📌 Overview

This tutorial helps you:

* Install Prometheus & Grafana
* Monitor EC2 or Linux servers
* Add custom exporters (Node Exporter)
* Build dashboards in Grafana
* Set alerts

---

## 🧰 Prerequisites

* 1 or more EC2 instances or local VMs
* Amazon Linux 2 / Ubuntu / RHEL (examples given for Amazon Linux 2)
* Basic Linux knowledge
* Internet access from the instance

---

## 🏗️ Step-by-Step Installation

### 🔧 1. Install Prometheus

```bash
# Create user & directories
sudo useradd --no-create-home --shell /bin/false prometheus
sudo mkdir /etc/prometheus /var/lib/prometheus

# Download Prometheus
cd /tmp
curl -s https://api.github.com/repos/prometheus/prometheus/releases/latest \
| grep browser_download_url | grep linux-amd64 \
| cut -d '"' -f 4 | wget -qi -

# Extract and install
tar xvf prometheus*.tar.gz
cd prometheus-*/
sudo cp prometheus promtool /usr/local/bin/
sudo cp -r consoles console_libraries /etc/prometheus

# Configuration
sudo tee /etc/prometheus/prometheus.yml > /dev/null <<EOF
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
EOF

# Systemd service
sudo tee /etc/systemd/system/prometheus.service > /dev/null <<EOF
[Unit]
Description=Prometheus Monitoring
After=network.target

[Service]
User=prometheus
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/var/lib/prometheus \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reexec
sudo systemctl enable --now prometheus
```

Access: `http://<EC2-IP>:9090`

---

### 🖥️ 2. Install Node Exporter (Linux Metrics)

```bash
# Download
cd /tmp
curl -s https://api.github.com/repos/prometheus/node_exporter/releases/latest \
| grep browser_download_url | grep linux-amd64 \
| cut -d '"' -f 4 | wget -qi -

# Extract and install
tar xvf node_exporter*.tar.gz
cd node_exporter-*/
sudo cp node_exporter /usr/local/bin/

# Systemd service
sudo tee /etc/systemd/system/node_exporter.service > /dev/null <<EOF
[Unit]
Description=Node Exporter

[Service]
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=default.target
EOF

sudo systemctl daemon-reexec
sudo systemctl enable --now node_exporter
```

Add to Prometheus config:

```yaml
  - job_name: 'node'
    static_configs:
      - targets: ['localhost:9100']
```

Reload Prometheus:

```bash
sudo systemctl restart prometheus
```

---

### 📊 3. Install Grafana

```bash
sudo yum install -y https://dl.grafana.com/oss/release/grafana-10.2.3-1.x86_64.rpm
sudo systemctl enable --now grafana-server
```

Access: `http://<EC2-IP>:3000`
Login: `admin / admin` → set new password.

---

## 📈 4. Configure Grafana with Prometheus

### ➕ Add Data Source

* Go to **Settings > Data Sources > Add data source**
* Choose **Prometheus**
* URL: `http://localhost:9090`
* Save & Test

---

### 📁 5. Import Dashboards

You can import dashboards from:

* [https://grafana.com/grafana/dashboards/](https://grafana.com/grafana/dashboards/)

Example Node Exporter Dashboard:
🔗 [Node Exporter Full – ID 1860](https://grafana.com/grafana/dashboards/1860)

Steps:

1. Click "+" > Import
2. Enter dashboard ID: `1860`
3. Choose Prometheus as the data source

---

## 🚨 6. Alerts (optional)

Basic alerting example:

```yaml
groups:
- name: example
  rules:
  - alert: InstanceDown
    expr: up == 0
    for: 1m
    labels:
      severity: critical
    annotations:
      summary: "Instance {{ $labels.instance }} is down"
```

Reload alert rules in Prometheus:

```bash
# Add in prometheus.yml
rule_files:
  - "alert.rules.yml"
```

---

## 🧪 7. Testing

* Stop node\_exporter: `sudo systemctl stop node_exporter`
* Check alerts in Prometheus
* View in Grafana panel

---

## ✅ Summary

| Tool          | Port | Role                         |
| ------------- | ---- | ---------------------------- |
| Prometheus    | 9090 | Metrics scraper & storage    |
| Node Exporter | 9100 | Exposes system metrics       |
| Grafana       | 3000 | Visualization and dashboards |

---

## 📦 Optional: Use Docker or EC2 AMI

* Docker Compose version of the stack
* Pre-built AMI with stack pre-installed
* Use Terraform to provision stack + exporter + dashboards

---

## 📚 Bonus Resources

* [Prometheus Docs](https://prometheus.io/docs/)
* [Grafana Docs](https://grafana.com/docs/)
* [Awesome Prometheus Alerts](https://awesome-prometheus-alerts.grep.to/)
* [Monitoring EC2 with Node Exporter](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/mon-scripts.html)

**drop-in, working “monitoring” module** you can drop into any repo and run. It ships Prometheus + Alertmanager + Grafana + Blackbox + Node Exporter, with rules, alerts, and Grafana provisioning. I’ve also included **“how to run”** and **“how to change config/code inside it”** so your team can iterate fast.

---

# Monitoring Module (Docker Compose)

```
monitoring/
├─ docker-compose.yml
├─ .env.example
├─ prometheus/
│  ├─ prometheus.yml
│  └─ rules/
│     ├─ recording.yml
│     └─ alerts.yml
├─ alertmanager/
│  └─ alertmanager.yml
├─ grafana/
│  └─ provisioning/
│     ├─ datasources/
│     │  └─ prometheus.yml
│     ├─ dashboards/
│     │  └─ dashboards.yml
│     ├─ alerting/
│     │  └─ highcpu.yaml
│     └─ notifiers/            (optional; empty by default)
│  └─ dashboards/
│     └─ infra.json
└─ Makefile
```

---

## 1) `docker-compose.yml`

```yaml
version: "3.9"

networks:
  monitoring:
    driver: bridge

volumes:
  prometheus_data:
  grafana_data:

services:

  prometheus:
    image: prom/prometheus:v2.53.0
    container_name: prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
      - "--web.enable-lifecycle"
      - "--storage.tsdb.retention.time=15d"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/rules:/etc/prometheus/rules:ro
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    networks: [monitoring]
    depends_on:
      - alertmanager
      - blackbox
    restart: unless-stopped

  alertmanager:
    image: prom/alertmanager:v0.27.0
    container_name: alertmanager
    command:
      - "--config.file=/etc/alertmanager/alertmanager.yml"
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
    ports:
      - "9093:9093"
    networks: [monitoring]
    restart: unless-stopped

  blackbox:
    image: prom/blackbox-exporter:v0.25.0
    container_name: blackbox
    ports:
      - "9115:9115"
    networks: [monitoring]
    restart: unless-stopped

  node_exporter:
    image: prom/node-exporter:v1.8.1
    container_name: node_exporter
    pid: host
    network_mode: "host"
    # mount host proc/sys for metrics
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro,rslave
    command:
      - "--path.procfs=/host/proc"
      - "--path.sysfs=/host/sys"
      - "--path.rootfs=/rootfs"
      - "--collector.filesystem.mount-points-exclude=^/(dev|proc|sys|run|var/lib/docker/.+|var/lib/containers/.+)"
    restart: unless-stopped

  grafana:
    image: grafana/grafana:10.4.3
    container_name: grafana
    environment:
      - GF_SECURITY_ADMIN_USER=${GF_SECURITY_ADMIN_USER:-admin}
      - GF_SECURITY_ADMIN_PASSWORD=${GF_SECURITY_ADMIN_PASSWORD:-admin}
      - GF_USERS_DEFAULT_THEME=light
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
      - ./grafana/dashboards:/var/lib/grafana/dashboards:ro
    ports:
      - "3000:3000"
    networks: [monitoring]
    depends_on:
      - prometheus
    restart: unless-stopped
```

---

## 2) `.env.example` (copy to `.env` and edit as needed)

```
GF_SECURITY_ADMIN_USER=admin
GF_SECURITY_ADMIN_PASSWORD=StrongP@ssw0rd
```

---

## 3) Prometheus config

### 3.1 `prometheus/prometheus.yml`

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 30s
  external_labels:
    cluster: "primary"

rule_files:
  - /etc/prometheus/rules/*.yml

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ['prometheus:9090']

  - job_name: node
    static_configs:
      # Local host via node_exporter (runs with host network)
      - targets: ['localhost:9100']

  - job_name: blackbox-http
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
          - https://grafana.com
          - https://prometheus.io
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox:9115
```

### 3.2 `prometheus/rules/recording.yml`

```yaml
groups:
- name: infra_recording
  interval: 30s
  rules:
    - record: instance:cpu_utilization:avg1m
      expr: 100 * (1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[1m])))

    - record: instance:mem_utilization:ratio
      expr: 1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes

    - record: instance:fs_usage:ratio
      expr: 1 - (node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"} / node_filesystem_size_bytes{fstype!~"tmpfs|overlay"})
```

### 3.3 `prometheus/rules/alerts.yml`

```yaml
groups:
- name: infra_alerts
  rules:
    - alert: InstanceDown
      expr: up == 0
      for: 2m
      labels: { severity: critical }
      annotations:
        summary: "Instance down: {{ $labels.instance }}"
        description: "Target has been down > 2m."

    - alert: HighCPU
      expr: instance:cpu_utilization:avg1m > 85
      for: 10m
      labels: { severity: warning }
      annotations:
        summary: "High CPU on {{ $labels.instance }}"
        description: "CPU > 85% for 10m."

    - alert: HighMemory
      expr: instance:mem_utilization:ratio > 0.9
      for: 10m
      labels: { severity: warning }
      annotations:
        summary: "High memory on {{ $labels.instance }}"
        description: "RAM > 90% for 10m."

    - alert: DiskWillFillSoon
      expr: predict_linear(node_filesystem_free_bytes{fstype!~"tmpfs|overlay"}[6h], 24*3600) < 0
      for: 10m
      labels: { severity: critical }
      annotations:
        summary: "Disk will fill within 24h on {{ $labels.instance }}"
        description: "Free bytes trend projects zero within 24h."
```

---

## 4) Alertmanager

### 4.1 `alertmanager/alertmanager.yml`

```yaml
route:
  receiver: default
  group_by: ['alertname','instance']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 2h
  routes:
    - matchers: ['severity="critical"']
      receiver: ops-critical
    - matchers: ['severity="warning"']
      receiver: ops-warning

receivers:
  - name: default
  - name: ops-critical
    # Example Slack webhook
    webhook_configs:
      - url: "https://hooks.slack.com/services/XXX/YYY/ZZZ"
  - name: ops-warning
    email_configs:
      - to: "devops@example.com"
        from: "alerts@example.com"
        smarthost: "smtp.example.com:587"
        auth_username: "alerts@example.com"
        auth_password: "<APP_PASSWORD>"

inhibit_rules:
  - source_matchers: ['severity="critical"']
    target_matchers: ['severity="warning"']
    equal: ['alertname','instance']
```

> Tip: If you don’t have Slack or SMTP yet, leave those blocks as-is; Alertmanager will still run.

---

## 5) Grafana provisioning

### 5.1 `grafana/provisioning/datasources/prometheus.yml`

```yaml
apiVersion: 1
datasources:
  - name: Prometheus
    uid: prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    jsonData:
      timeInterval: 15s
```

### 5.2 `grafana/provisioning/dashboards/dashboards.yml`

```yaml
apiVersion: 1
providers:
  - name: "infra-boards"
    folder: "Infrastructure"
    type: file
    disableDeletion: false
    updateIntervalSeconds: 30
    options:
      path: /var/lib/grafana/dashboards
```

### 5.3 `grafana/provisioning/alerting/highcpu.yaml` (optional Grafana rule)

```yaml
apiVersion: 1
groups:
  - orgId: 1
    name: infra
    interval: 1m
    rules:
      - uid: highcpu-grafana
        title: High CPU (Grafana)
        condition: C
        data:
          - refId: A
            datasourceUid: prometheus
            relativeTimeRange: { from: 600, to: 0 }
            model:
              expr: avg(instance:cpu_utilization:avg1m) > 85
              instant: true
              refId: A
          - refId: B
            datasourceUid: __expr__
            model: { type: reduce, reducer: last, expression: A, refId: B }
          - refId: C
            datasourceUid: __expr__
            model:
              type: threshold
              expression: B
              conditions:
                - evaluator: { type: gt, params: [0] }
        for: 10m
        annotations: { runbook_url: "https://runbooks.example.com/highcpu" }
        labels: { severity: warning }
        isPaused: false
```

### 5.4 `grafana/dashboards/infra.json` (minimal ready dashboard)

```json
{
  "title": "Infra – Nodes Overview",
  "schemaVersion": 39,
  "version": 1,
  "timezone": "browser",
  "panels": [
    {
      "type": "timeseries",
      "title": "CPU % (avg1m)",
      "targets": [{ "expr": "instance:cpu_utilization:avg1m", "legendFormat": "{{instance}}" }],
      "gridPos": { "h": 8, "w": 12, "x": 0, "y": 0 }
    },
    {
      "type": "timeseries",
      "title": "Memory % used",
      "targets": [{ "expr": "100 * instance:mem_utilization:ratio", "legendFormat": "{{instance}}" }],
      "gridPos": { "h": 8, "w": 12, "x": 12, "y": 0 }
    },
    {
      "type": "timeseries",
      "title": "Disk usage % (all filesystems)",
      "targets": [{ "expr": "100 * instance:fs_usage:ratio", "legendFormat": "{{instance}}" }],
      "gridPos": { "h": 8, "w": 24, "x": 0, "y": 8 }
    }
  ]
}
```

---

## 6) Makefile (quality-of-life)

```makefile
.PHONY: up down logs reload promql targets rules alerts

up:
\tdocker compose up -d

down:
\tdocker compose down -v

logs:
\tdocker compose logs -f

reload: ## hot-reload Prometheus rules
\tcurl -X POST http://localhost:9090/-/reload

targets:
\tcurl -s http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | {job: .labels.job, instance: .labels.instance, health: .health}'

rules:
\tcurl -s http://localhost:9090/api/v1/rules | jq '.data.groups[].name'

alerts:
\tcurl -s http://localhost:9090/api/v1/alerts | jq
```

---

# How to Run

```bash
cd monitoring
cp .env.example .env          # optional: update admin password
docker compose up -d
# Wait ~10–20s
open http://localhost:3000    # Grafana (user/pass from .env)
open http://localhost:9090    # Prometheus
open http://localhost:9093    # Alertmanager
open http://localhost:9115    # Blackbox
```

**Validate quickly**

```bash
# Prometheus targets must be UP
make targets

# Rules loaded?
make rules

# Fire a test alert: stop node_exporter for 3–5 min (you'll see InstanceDown)
docker stop node_exporter
sleep 180
make alerts
docker start node_exporter
```

---

# How to Configure / Modify “Code” Inside This Module

### A) Add scrape targets

* Edit `prometheus/prometheus.yml` → under `scrape_configs` add your service:

```yaml
- job_name: myapp
  static_configs:
    - targets: ['app1.internal:8080','app2.internal:8080']
```

* Apply without restart:

```bash
make reload
```

### B) Use service discovery (EC2/Kubernetes)

If you’re on EC2/K8s, replace the `myapp` block with SD (ping me if you want the exact `ec2_sd_configs` or `kubernetes_sd_configs` tuned for your tags/namespaces).

### C) Add/adjust recording & alert rules

* Put new rules in `prometheus/rules/recording.yml` or `alerts.yml`.
* Reload Prometheus:

```bash
make reload
```

* Validate in Prometheus UI → **Status → Rules**.

### D) Change Blackbox probes

* Edit the `targets` list in the `blackbox-http` job.
* `make reload`.

### E) Provision Grafana dashboards as code

* Drop a new JSON in `grafana/dashboards/` and reference it via `dashboards.yml` (already set to load the whole folder).
* Grafana will discover changes automatically (updateIntervalSeconds=30) or restart Grafana:

```bash
docker restart grafana
```

### F) Configure notifications

* **Alertmanager**: set Slack or Email in `alertmanager/alertmanager.yml`.
* **Grafana** (optional): add contact points via UI (Alerting → Contact points) or add files in `grafana/provisioning/notifiers/` (if you prefer code-first).

### G) Secure for production

* Reverse proxy + TLS (nginx/ALB) in front of Grafana/Prometheus.
* Set Grafana OIDC/Entra ID SSO (env vars or `grafana.ini`).
* Network-restrict Prometheus/Alertmanager UIs.

---

# Optional: Add Your App’s Metrics (Python example)

If your Flask app isn’t instrumented yet:

```bash
pip install prometheus_client
```

```python
# app.py
from flask import Flask, Response
from prometheus_client import Counter, generate_latest, CONTENT_TYPE_LATEST

app = Flask(__name__)
REQUESTS = Counter('http_requests_total', 'Total HTTP requests', ['path'])

@app.before_request
def before():
    from flask import request
    REQUESTS.labels(request.path).inc()

@app.route('/metrics')
def metrics():
    return Response(generate_latest(), mimetype=CONTENT_TYPE_LATEST)

@app.route('/')
def index():
    return "OK", 200

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

Then add to Prometheus:

```yaml
- job_name: flask
  static_configs:
    - targets: ['host.docker.internal:8080']  # or your service DNS/IP
```

`make reload` and build panels in Grafana using `http_requests_total`.

---



