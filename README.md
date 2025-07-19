# ec2 | t3.medium | 20GB SSD | key.pem

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


