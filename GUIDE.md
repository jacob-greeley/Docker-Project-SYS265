# Prometheus Monitoring Stack — Complete Deployment Guide

A detailed, step-by-step walkthrough for deploying the full Prometheus monitoring stack on Ubuntu using Docker Compose. Every command is included — nothing is assumed.

> **Target OS:** Ubuntu 22.04 / 24.04 LTS  
> **Time to complete:** ~30 minutes  
> **Skill level:** Beginner to Intermediate

## Two Ways to Use This Guide

**The fast way (recommended):** All configuration files are packaged in the release zip on this repository. Download and deploy:

```bash
# Download the latest release zip from GitHub
wget https://github.com/jacob-greeley/Docker-Project-SYS265/releases/latest/download/Prometheus-monitoring-stack.zip

# Unzip and enter the directory
mkdir -p ~/Docker-Project-SYS265 && cd ~/Docker-Project-SYS265
unzip ~/Prometheus-monitoring-stack.zip

# Configure and deploy
cp .env.example .env
vim .env                   # Set your passwords
docker compose up -d
```

Then skip to [Step 12 — Deploy the Stack](#step-12--deploy-the-stack) for verification.

**The learning way:** Follow every step below to understand what each file does and build it from scratch.

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Step 1 — Install Docker Engine on Ubuntu](#step-1--install-docker-engine-on-ubuntu)
- [Step 2 — Open Firewall Ports (UFW)](#step-2--open-firewall-ports-ufw)
- [Step 3 — Create the Project Directory Structure](#step-3--create-the-project-directory-structure)
- [Step 4 — Create the Environment Variables File](#step-4--create-the-environment-variables-file)
- [Step 5 — Create the Docker Compose File](#step-5--create-the-docker-compose-file)
- [Step 6 — Create the Prometheus Configuration](#step-6--create-the-prometheus-configuration)
- [Step 7 — Create the Alert Rules](#step-7--create-the-alert-rules)
- [Step 8 — Create the Alertmanager Configuration](#step-8--create-the-alertmanager-configuration)
- [Step 9 — Create the Grafana Provisioning Files](#step-9--create-the-grafana-provisioning-files)
- [Step 10 — Create the Nginx Configuration](#step-10--create-the-nginx-configuration)
- [Step 11 — Create the MySQL Initialization Script](#step-11--create-the-mysql-initialization-script)
- [Step 12 — Deploy the Stack](#step-12--deploy-the-stack)
- [Step 13 — Verify All Services Are Running](#step-13--verify-all-services-are-running)
- [Step 14 — Access the Dashboards](#step-14--access-the-dashboards)
- [Step 15 — Configure Grafana](#step-15--configure-grafana)
- [Day-to-Day Commands Reference](#day-to-day-commands-reference)
- [Adding More Monitoring Targets](#adding-more-monitoring-targets)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

Before starting, confirm your Ubuntu system meets these requirements:

| Requirement | Minimum | Recommended |
|---|---|---|
| **Operating System** | Ubuntu 22.04 LTS | Ubuntu 24.04 LTS |
| **Docker Engine** | 24.0+ | 27.0+ |
| **Docker Compose** | v2.20+ (bundled with Docker) | v2.30+ |
| **RAM** | 4 GB | 8 GB |
| **Free Disk Space** | 10 GB | 20 GB+ |
| **Ports Available** | 3000, 8080, 9090, 9093 | — |

> ⚠️ **Do NOT** install Docker from Ubuntu's default `apt` repository (the `docker.io` package). It is outdated and does not include the Compose plugin. Always use Docker's official repository as shown in Step 1.

> ⚠️ Ubuntu 22.04+ defaults to **cgroup v2**. This project handles that automatically — no manual cgroup configuration is needed.

---

## Step 1 — Install Docker Engine on Ubuntu

Docker Compose V2 is bundled with Docker Engine. We install Docker from the official Docker repository to get the latest version.

### 1a. Remove Conflicting Packages

Remove any old or conflicting Docker packages that may have been installed from Ubuntu's default repositories:

```bash
sudo apt remove docker docker-engine docker.io containerd runc docker-compose 2>/dev/null
```

### 1b. Install Prerequisites

Update the package index and install the packages needed to add Docker's repository:

```bash
sudo apt update
sudo apt install -y ca-certificates curl
```

### 1c. Add Docker's Official GPG Key

Create the keyrings directory and download Docker's signing key:

```bash
sudo install -m 0755 -d /etc/apt/keyrings

sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg \
  -o /etc/apt/keyrings/docker.asc

sudo chmod a+r /etc/apt/keyrings/docker.asc
```

### 1d. Add the Docker Repository

Add the official Docker repository using the current DEB822 format:

```bash
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

### 1e. Install Docker Engine + Compose Plugin

Update the package index again (to pick up the new Docker repo) and install:

```bash
sudo apt update

sudo apt install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
```

### 1f. Enable Docker on Boot

Ensure Docker starts automatically when the system boots:

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

### 1g. Run Docker Without sudo

Add your user to the `docker` group so you don't need `sudo` for every Docker command:

```bash
sudo usermod -aG docker $USER
```

> ⚠️ **You MUST log out and log back in** (or run `newgrp docker`) for this change to take effect.

### 1h. Configure Docker for cgroup v2 (Ubuntu 22.04+)

Ubuntu 22.04+ uses cgroup v2 by default, which requires Docker to use the host cgroup namespace for monitoring containers like cAdvisor. Create or edit the Docker daemon config:

```bash
sudo mkdir -p /etc/docker

sudo tee /etc/docker/daemon.json <<EOF
{
  "default-cgroupns-mode": "host"
}
EOF

sudo systemctl restart docker
```

> ⚠️ This sets the **default** cgroup namespace mode to `host` for all containers. This is safe for lab environments. In production, you may prefer to set this per-container only.

### 1i. Verify the Installation

Confirm Docker and Compose are installed correctly:

```bash
docker --version
# Expected output: Docker version 27.x.x or higher

docker compose version
# Expected output: Docker Compose version v2.30.x or higher

docker run --rm hello-world
# Expected output: "Hello from Docker!" message
```

> ✅ Always use `docker compose` (with a space). The hyphenated `docker-compose` is the legacy V1 binary and is no longer maintained.

---

## Step 2 — Open Firewall Ports (UFW)

If Ubuntu's UFW firewall is enabled, open the ports needed by the monitoring stack.

### 2a. Check if UFW is Active

```bash
sudo ufw status
```

If UFW shows `inactive`, you can skip to Step 3. If it shows `active`, continue below.

### 2b. Allow the Required Ports

```bash
sudo ufw allow 3000/tcp comment "Grafana"
sudo ufw allow 8080/tcp comment "Nginx web server"
sudo ufw allow 9090/tcp comment "Prometheus"
sudo ufw allow 9093/tcp comment "Alertmanager"
```

### 2c. Reload UFW

```bash
sudo ufw reload
```

### 2d. Verify the Rules

```bash
sudo ufw status numbered
```

You should see entries for ports 3000, 8080, 9090, and 9093 in the output.

> ⚠️ Docker manipulates iptables directly and can bypass UFW rules for published ports. For strict network control in production, bind services to localhost (e.g., `127.0.0.1:9090:9090`) and place a reverse proxy in front.

---

## Step 3 — Create the Project Directory Structure

### 3a. Create the Root Directory

```bash
mkdir -p ~/Docker-Project-SYS265
cd ~/Docker-Project-SYS265
```

### 3b. Create All Subdirectories

```bash
mkdir -p prometheus
mkdir -p alertmanager
mkdir -p grafana/provisioning/datasources
mkdir -p grafana/provisioning/dashboards
mkdir -p grafana/dashboards
mkdir -p nginx/conf.d
mkdir -p nginx/html
mkdir -p mysql/init
```

### 3c. Verify the Structure

```bash
find . -type d | sort
```

Expected output:

```
.
./alertmanager
./grafana
./grafana/dashboards
./grafana/provisioning
./grafana/provisioning/dashboards
./grafana/provisioning/datasources
./mysql
./mysql/init
./nginx
./nginx/conf.d
./nginx/html
./prometheus
```

---

## Step 4 — Create the Environment Variables File

This file stores passwords and configuration values. Docker Compose reads it automatically from the project root.

### 4a. Create the File

```bash
vim .env
```

### 4b. Paste the Following Content

Replace the default passwords with your own secure values:

```env
# Grafana
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=changeme

# MySQL
MYSQL_ROOT_PASSWORD=rootpassword
MYSQL_DATABASE=monitoring_db
MYSQL_USER=appuser
MYSQL_PASSWORD=apppassword
MYSQL_EXPORTER_PASSWORD=exporterpass
```

### 4c. Save and Exit

Press `Esc`, type `:wq`, and press `Enter`.

> ⚠️ **NEVER commit the `.env` file to git.** It contains secrets. The `.gitignore` file in the project excludes it automatically.

---

## Step 5 — Create the Docker Compose File

This is the main orchestration file. It defines all 9 services (including the Nginx Exporter), 3 networks, and 4 persistent volumes.

### 5a. Create the File

```bash
vim compose.yaml
```

### 5b. Paste the Following Content

```yaml
name: prometheus-monitoring-stack

services:

  # ---------------------------------------------------------------------------
  # CORE MONITORING
  # ---------------------------------------------------------------------------

  prometheus:
    image: prom/prometheus:v2.53.5
    container_name: prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - ./prometheus/alert_rules.yml:/etc/prometheus/alert_rules.yml:ro
      - prometheus_data:/prometheus
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
      - "--storage.tsdb.retention.time=15d"
      - "--web.enable-lifecycle"
      - "--web.enable-admin-api"
    networks:
      - monitoring
      - backend
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:9090/-/healthy"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 10s

  grafana:
    image: grafana/grafana:11.4.0
    container_name: grafana
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=${GRAFANA_ADMIN_USER:-admin}
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_ADMIN_PASSWORD:-changeme}
      - GF_USERS_ALLOW_SIGN_UP=false
      - GF_SERVER_ROOT_URL=http://localhost:3000
      - GF_INSTALL_PLUGINS=grafana-clock-panel
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
      - ./grafana/dashboards:/var/lib/grafana/dashboards:ro
    networks:
      - monitoring
    depends_on:
      prometheus:
        condition: service_healthy
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:3000/api/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s

  alertmanager:
    image: prom/alertmanager:v0.27.0
    container_name: alertmanager
    restart: unless-stopped
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
      - alertmanager_data:/alertmanager
    command:
      - "--config.file=/etc/alertmanager/alertmanager.yml"
      - "--storage.path=/alertmanager"
    networks:
      - monitoring
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:9093/-/healthy"]
      interval: 30s
      timeout: 10s
      retries: 3

  # ---------------------------------------------------------------------------
  # EXPORTERS (Metric Collectors)
  # ---------------------------------------------------------------------------

  node-exporter:
    image: prom/node-exporter:v1.8.2
    container_name: node-exporter
    restart: unless-stopped
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - "--path.procfs=/host/proc"
      - "--path.sysfs=/host/sys"
      - "--path.rootfs=/rootfs"
      - "--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)"
    networks:
      - monitoring
    pid: host

  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.49.1
    container_name: cadvisor
    restart: unless-stopped
    privileged: true
    cgroup: host
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
      - /sys/fs/cgroup:/sys/fs/cgroup:ro
    devices:
      - /dev/kmsg
    command:
      - "--docker_only=true"
      - "--housekeeping_interval=15s"
      - "--disable_metrics=referenced_memory,tcp,udp,sched,hugetlb,resctrl"
    networks:
      - monitoring

  mysql-exporter:
    image: prom/mysqld-exporter:v0.16.0
    container_name: mysql-exporter
    restart: unless-stopped
    environment:
      MYSQLD_EXPORTER_PASSWORD: ${MYSQL_EXPORTER_PASSWORD:-exporterpass}
    command:
      - "--mysqld.username=exporter"
      - "--mysqld.address=mysql:3306"
    networks:
      - monitoring
      - backend
    depends_on:
      mysql:
        condition: service_healthy

  nginx-exporter:
    image: nginx/nginx-prometheus-exporter:1.4.0
    container_name: nginx-exporter
    restart: unless-stopped
    command:
      - "--nginx.scrape-uri=http://nginx:80/stub_status"
    networks:
      - monitoring
    depends_on:
      - nginx

  # ---------------------------------------------------------------------------
  # APPLICATION SERVICES (Web + Database)
  # ---------------------------------------------------------------------------

  nginx:
    image: nginx:1.27-alpine
    container_name: nginx-web
    restart: unless-stopped
    ports:
      - "8080:80"
    volumes:
      - ./nginx/conf.d/default.conf:/etc/nginx/conf.d/default.conf:ro
      - ./nginx/html:/usr/share/nginx/html:ro
    networks:
      - monitoring
      - frontend
    depends_on:
      - prometheus
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:80/"]
      interval: 30s
      timeout: 10s
      retries: 3

  mysql:
    image: mysql:8.0
    container_name: mysql-db
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD:-rootpassword}
      MYSQL_DATABASE: ${MYSQL_DATABASE:-monitoring_db}
      MYSQL_USER: ${MYSQL_USER:-appuser}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD:-apppassword}
    volumes:
      - mysql_data:/var/lib/mysql
      - ./mysql/init:/docker-entrypoint-initdb.d:ro
    networks:
      - backend
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p${MYSQL_ROOT_PASSWORD:-rootpassword}"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 60s

networks:
  monitoring:
    driver: bridge
    name: monitoring-net
  frontend:
    driver: bridge
    name: frontend-net
  backend:
    driver: bridge
    name: backend-net

volumes:
  prometheus_data:
    name: prometheus-data
  grafana_data:
    name: grafana-data
  alertmanager_data:
    name: alertmanager-data
  mysql_data:
    name: mysql-data
```

### 5c. Save and Exit

Press `Esc`, type `:wq`, and press `Enter`.

### Key Things to Know About This File

- **No `version` key** — The `version: '3.8'` property is obsolete in Docker Compose V2+. Omitting it avoids deprecation warnings.
- **`cgroup: host` on cAdvisor** — Required for Ubuntu 22.04+ which uses cgroup v2 by default. Without this, cAdvisor fails with "mountpoint for cpu not found".
- **Nginx Exporter** — Nginx does not natively expose Prometheus metrics. The `nginx-exporter` container scrapes Nginx's `stub_status` endpoint and re-exposes the data as Prometheus metrics on port 9113. Prometheus scrapes the exporter, not Nginx directly.
- **Health checks** — Prometheus, Grafana, Nginx, and MySQL have health checks. Dependent services (like Grafana depending on Prometheus) wait until the dependency is healthy before starting.
- **`:ro` mounts** — Configuration files are mounted read-only to prevent containers from modifying your host files.
- **Named volumes** — Data for Prometheus, Grafana, MySQL, and Alertmanager persists across container restarts. Data is only deleted if you explicitly run `docker compose down -v`.

---

## Step 6 — Create the Prometheus Configuration

This file tells Prometheus what to monitor (scrape), how often, and where to send alerts.

### 6a. Create the File

```bash
vim prometheus/prometheus.yml
```

### 6b. Paste the Following Content

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  scrape_timeout: 10s

alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - "alertmanager:9093"

rule_files:
  - "alert_rules.yml"

scrape_configs:

  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]
        labels:
          instance: "prometheus-server"

  - job_name: "node-exporter"
    static_configs:
      - targets: ["node-exporter:9100"]
        labels:
          instance: "docker-host"

  - job_name: "cadvisor"
    static_configs:
      - targets: ["cadvisor:8080"]
        labels:
          instance: "docker-containers"

  - job_name: "mysql-exporter"
    static_configs:
      - targets: ["mysql-exporter:9104"]
        labels:
          instance: "mysql-db"

  - job_name: "nginx"
    static_configs:
      - targets: ["nginx-exporter:9113"]
        labels:
          instance: "nginx-web"

  - job_name: "windows"
    static_configs:
      - targets: ["10.0.5.5:9182"]
        labels:
          instance: "ad01-jacob"
          os: "windows"
```

### 6c. Save and Exit

Press `Esc`, type `:wq`, and press `Enter`.

### How This Works

- **`scrape_interval: 15s`** — Prometheus pulls metrics from every target every 15 seconds.
- **`scrape_configs`** — Each entry defines a monitoring target. Internal targets use **container names** as hostnames (like `node-exporter:9100`) because they share the Docker `monitoring-net` network. External targets use IP addresses (like `10.0.5.5:9182` for ad01-jacob).
- **Nginx metrics** — Prometheus scrapes `nginx-exporter:9113`, not Nginx directly. The Nginx Exporter container translates Nginx's `stub_status` page into Prometheus metrics format.
- **Windows target** — The `windows` job scrapes `windows_exporter` running on ad01-jacob. This requires `windows_exporter` to be installed on the Windows server and port 9182 to be open in its firewall.
- **`rule_files`** — Points to the alert rules file we create in the next step.

> ✅ To add more servers later, just add new entries under `scrape_configs` and reload: `curl -X POST http://localhost:9090/-/reload`

---

## Step 7 — Create the Alert Rules

Alert rules define conditions that trigger notifications (e.g., "CPU above 85% for 5 minutes").

### 7a. Create the File

```bash
vim prometheus/alert_rules.yml
```

### 7b. Paste the Following Content

```yaml
groups:

  - name: infrastructure
    rules:

      - alert: TargetDown
        expr: up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Target {{ $labels.job }} is down"
          description: "{{ $labels.instance }} of job {{ $labels.job }} has been down for more than 1 minute."

      - alert: HighCpuUsage
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU usage on {{ $labels.instance }}"
          description: "CPU usage is above 85% for more than 5 minutes (current: {{ $value | printf \"%.1f\" }}%)."

      - alert: HighMemoryUsage
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) * 100 > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage on {{ $labels.instance }}"
          description: "Memory usage is above 85% for more than 5 minutes (current: {{ $value | printf \"%.1f\" }}%)."

      - alert: DiskSpaceLow
        expr: (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100 < 15
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Low disk space on {{ $labels.instance }}"
          description: "Disk space is below 15% on root partition (current: {{ $value | printf \"%.1f\" }}% free)."

  - name: containers
    rules:

      - alert: ContainerHighCpu
        expr: (sum by(name) (rate(container_cpu_usage_seconds_total{name!=""}[5m])) * 100) > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Container {{ $labels.name }} high CPU usage"
          description: "Container {{ $labels.name }} CPU usage is above 80% (current: {{ $value | printf \"%.1f\" }}%)."

      - alert: ContainerHighMemory
        expr: (container_memory_usage_bytes{name!=""} / container_spec_memory_limit_bytes{name!=""} * 100) > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Container {{ $labels.name }} high memory usage"
          description: "Container {{ $labels.name }} memory usage is above 85%."

  - name: mysql
    rules:

      - alert: MySQLDown
        expr: mysql_up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "MySQL instance is down"
          description: "MySQL has been unreachable for more than 1 minute."

      - alert: MySQLTooManyConnections
        expr: mysql_global_status_threads_connected / mysql_global_variables_max_connections * 100 > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "MySQL too many connections"
          description: "MySQL connections are above 80% of max (current: {{ $value | printf \"%.1f\" }}%)."
```

### 7c. Save and Exit

Press `Esc`, type `:wq`, and press `Enter`.

### Alert Summary

| Alert | Condition | Severity |
|---|---|---|
| TargetDown | Any monitored target unreachable > 1 min | Critical |
| HighCpuUsage | Host CPU > 85% for 5 min | Warning |
| HighMemoryUsage | Host memory > 85% for 5 min | Warning |
| DiskSpaceLow | Root disk < 15% free for 5 min | Warning |
| ContainerHighCpu | Any container CPU > 80% for 5 min | Warning |
| ContainerHighMemory | Any container memory > 85% for 5 min | Warning |
| MySQLDown | MySQL unreachable > 1 min | Critical |
| MySQLTooManyConnections | MySQL connections > 80% of max for 5 min | Warning |

---

## Step 8 — Create the Alertmanager Configuration

Alertmanager receives alerts from Prometheus and routes them to notification receivers (email, Slack, webhooks, etc.).

### 8a. Create the File

```bash
vim alertmanager/alertmanager.yml
```

### 8b. Paste the Following Content

```yaml
global:
  resolve_timeout: 5m

route:
  group_by: ["alertname", "severity"]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  receiver: "default"

  routes:
    - match:
        severity: critical
      receiver: "critical"
      group_wait: 10s

receivers:
  - name: "default"
    # To enable email notifications, uncomment and configure:
    # email_configs:
    #   - to: "you@example.com"
    #     from: "alertmanager@example.com"
    #     smarthost: "smtp.example.com:587"
    #     auth_username: "you@example.com"
    #     auth_password: "your-smtp-password"

  - name: "critical"
    # To enable Slack notifications, uncomment and configure:
    # slack_configs:
    #   - api_url: "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
    #     channel: "#alerts"
    #     send_resolved: true

inhibit_rules:
  - source_match:
      severity: "critical"
    target_match:
      severity: "warning"
    equal: ["alertname", "instance"]
```

### 8c. Save and Exit

Press `Esc`, type `:wq`, and press `Enter`.

> ℹ️ By default, alerts are received but **not forwarded anywhere**. To enable notifications, uncomment and configure the `email_configs` or `slack_configs` sections shown above.

---

## Step 9 — Create the Grafana Provisioning Files

These files auto-configure Grafana on first boot so that Prometheus is registered as a data source and dashboards load automatically. No manual setup needed.

### 9a. Create the Datasource Configuration

```bash
vim grafana/provisioning/datasources/prometheus.yml
```

Paste:

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: true
    jsonData:
      timeInterval: "15s"
```

Save and exit — press `Esc`, type `:wq`, press `Enter`.

### 9b. Create the Dashboard Provisioning Configuration

```bash
vim grafana/provisioning/dashboards/dashboards.yml
```

Paste:

```yaml
apiVersion: 1

providers:
  - name: "Default"
    orgId: 1
    folder: ""
    type: file
    disableDeletion: false
    editable: true
    options:
      path: /var/lib/grafana/dashboards
      foldersFromFilesStructure: false
```

Save and exit — press `Esc`, type `:wq`, press `Enter`.

### 9c. Create the Dashboard JSON File

This is a pre-built dashboard with CPU, memory, disk, network, and target status panels. Because the JSON is long, the easiest method is to download it directly from the project:

```bash
# If you downloaded the release zip, the file is already at grafana/dashboards/node-overview.json

# If building manually, copy it from the release zip or create it per the README
```

> ✅ Alternatively, copy the `node-overview.json` file from the project zip into `grafana/dashboards/`.

---

## Step 10 — Create the Nginx Configuration

Nginx serves as the **web container** and exposes a `stub_status` endpoint that Prometheus scrapes for connection metrics.

### 10a. Create the Virtual Host Configuration

```bash
vim nginx/conf.d/default.conf
```

Paste:

```nginx
server {
    listen 80;
    server_name localhost;

    # Main site
    location / {
        root   /usr/share/nginx/html;
        index  index.html;
    }

    # Nginx stub_status endpoint for Prometheus scraping
    location /stub_status {
        stub_status;
        allow 172.0.0.0/8;   # Docker network range
        allow 10.0.0.0/8;    # Private ranges
        allow 192.168.0.0/16;
        deny all;
    }

    # Health check endpoint
    location /health {
        access_log off;
        return 200 "OK\n";
        add_header Content-Type text/plain;
    }
}
```

Save and exit — press `Esc`, type `:wq`, press `Enter`.

### 10b. Create the Landing Page

```bash
vim nginx/html/index.html
```

Paste:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Prometheus Monitoring Stack</title>
    <style>
        body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
               max-width: 800px; margin: 40px auto; padding: 0 20px;
               background: #0d1117; color: #c9d1d9; }
        h1 { color: #58a6ff; border-bottom: 1px solid #30363d; padding-bottom: 12px; }
        a { color: #58a6ff; text-decoration: none; }
        a:hover { text-decoration: underline; }
        .card { background: #161b22; border: 1px solid #30363d;
                border-radius: 6px; padding: 16px; margin: 12px 0; }
        .card h3 { margin-top: 0; color: #f0f6fc; }
        code { background: #1f2937; padding: 2px 6px; border-radius: 4px; font-size: 0.9em; }
    </style>
</head>
<body>
    <h1>&#x1F4CA; Prometheus Monitoring Stack</h1>
    <p>All services are running. Use the links below to access each component.</p>
    <div class="card">
        <h3>Grafana &mdash; Dashboards &amp; Visualization</h3>
        <p><a href="http://localhost:3000">http://localhost:3000</a> &mdash;
           Default login: <code>admin</code> / <code>changeme</code></p>
    </div>
    <div class="card">
        <h3>Prometheus &mdash; Metrics &amp; Queries</h3>
        <p><a href="http://localhost:9090">http://localhost:9090</a> &mdash; PromQL query interface</p>
    </div>
    <div class="card">
        <h3>Alertmanager &mdash; Alert Routing</h3>
        <p><a href="http://localhost:9093">http://localhost:9093</a> &mdash; Active alerts and silences</p>
    </div>
    <div class="card">
        <h3>This Page &mdash; Nginx Web Server</h3>
        <p>Served on <a href="http://localhost:8080">http://localhost:8080</a> &mdash;
           Monitored via <code>stub_status</code></p>
    </div>
</body>
</html>
```

Save and exit — press `Esc`, type `:wq`, press `Enter`.

---

## Step 11 — Create the MySQL Initialization Script

This SQL script runs automatically when the MySQL container starts **for the first time**. It creates the Prometheus exporter user and a sample table.

### 11a. Create the File

```bash
vim mysql/init/01-init.sql
```

### 11b. Paste the Following Content

```sql
-- Create the exporter user for mysqld_exporter with minimal privileges
CREATE USER IF NOT EXISTS 'exporter'@'%' IDENTIFIED BY 'exporterpass';
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'%';
FLUSH PRIVILEGES;

-- Sample application schema
USE monitoring_db;

CREATE TABLE IF NOT EXISTS app_events (
    id          INT AUTO_INCREMENT PRIMARY KEY,
    event_type  VARCHAR(50)  NOT NULL,
    message     TEXT,
    created_at  TIMESTAMP    DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO app_events (event_type, message) VALUES
    ('startup', 'Monitoring stack initialized'),
    ('info', 'MySQL database is ready for connections');
```

### 11c. Save and Exit

Press `Esc`, type `:wq`, and press `Enter`.

> ⚠️ If you changed `MYSQL_EXPORTER_PASSWORD` in `.env`, update the password (`'exporterpass'`) in this file to match.

---

## Step 12 — Deploy the Stack

Everything is configured. Time to pull the images and start all services.

### 12a. Make Sure You Are in the Project Directory

```bash
cd ~/Docker-Project-SYS265
ls -la
```

You should see `compose.yaml`, `.env`, and all the subdirectories (`prometheus/`, `grafana/`, etc.).

### 12b. Pull All Docker Images

```bash
docker compose pull
```

This downloads all 8 container images. Expect approximately **2 GB** of data depending on your connection.

### 12c. Start All Services

```bash
docker compose up -d
```

The `-d` flag runs containers in detached (background) mode. Expected output:

```
[+] Running 12/12
 ✔ Network monitoring-net    Created
 ✔ Network frontend-net      Created
 ✔ Network backend-net       Created
 ✔ Volume  prometheus-data   Created
 ✔ Volume  grafana-data      Created
 ✔ Volume  alertmanager-data Created
 ✔ Volume  mysql-data        Created
 ✔ Container prometheus      Started
 ✔ Container mysql-db        Started
 ✔ Container node-exporter   Started
 ✔ Container cadvisor        Started
 ✔ Container alertmanager    Started
 ✔ Container mysql-exporter  Started
 ✔ Container nginx-exporter  Started
 ✔ Container grafana         Started
 ✔ Container nginx-web       Started
```

### 12d. Wait for Health Checks

MySQL takes the longest to initialize (up to 60 seconds on first run). Wait briefly, then proceed to verification.

```bash
# Wait 30 seconds for services to stabilize
sleep 30
```

---

## Step 13 — Verify All Services Are Running

### 13a. Check Container Status

```bash
docker compose ps
```

Expected output (all should show `Up` or `Up (healthy)`):

```
NAME              IMAGE                                    STATUS                   PORTS
alertmanager      prom/alertmanager:v0.27.0                Up (healthy)             0.0.0.0:9093->9093/tcp
cadvisor          gcr.io/cadvisor/cadvisor:v0.49.1         Up                       8080/tcp
grafana           grafana/grafana:11.4.0                   Up (healthy)             0.0.0.0:3000->3000/tcp
mysql-db          mysql:8.0                                Up (healthy)             3306/tcp
mysql-exporter    prom/mysqld-exporter:v0.16.0             Up                       9104/tcp
nginx-exporter    nginx/nginx-prometheus-exporter:1.4.0    Up                       9113/tcp
nginx-web         nginx:1.27-alpine                        Up (healthy)             0.0.0.0:8080->80/tcp
node-exporter     prom/node-exporter:v1.8.2                Up                       9100/tcp
prometheus        prom/prometheus:v2.53.5                   Up (healthy)             0.0.0.0:9090->9090/tcp
```

### 13b. Verify Prometheus Is Scraping Targets

```bash
curl -s http://localhost:9090/api/v1/targets | python3 -m json.tool | grep -E '"health"|"job"'
```

All targets should show `"health": "up"`. You can also check this visually in the browser:

```
http://localhost:9090/targets
```

All 6 targets should have a green **UP** badge.

> **Note:** The `windows` target (`10.0.5.5:9182`) will only show UP if `windows_exporter` is running on ad01-jacob and the firewall allows traffic on port 9182 from the Docker host.

### 13c. Check Logs if Anything Looks Wrong

```bash
# View logs for a specific service
docker compose logs prometheus
docker compose logs grafana
docker compose logs mysql-db
docker compose logs cadvisor

# View last 50 lines from ALL services
docker compose logs --tail=50

# Follow logs in real time (Ctrl+C to stop)
docker compose logs -f
```

### 13d. Check Resource Usage

```bash
docker stats --no-stream
```

This shows CPU, memory, and network usage for each container.

---

## Step 14 — Access the Dashboards

Open a web browser and visit these URLs. If you are accessing from another machine on your network, replace `localhost` with the Ubuntu server's IP address.

| Service | URL | Login |
|---|---|---|
| **Grafana** | [http://localhost:3000](http://localhost:3000) | `admin` / `changeme` (or your `.env` values) |
| **Prometheus** | [http://localhost:9090](http://localhost:9090) | No login required |
| **Alertmanager** | [http://localhost:9093](http://localhost:9093) | No login required |
| **Nginx Landing Page** | [http://localhost:8080](http://localhost:8080) | No login required |

> ✅ To find your server's IP address: `hostname -I | awk '{print $1}'`

---

## Step 15 — Configure Grafana

### 15a. Log In

1. Open [http://localhost:3000](http://localhost:3000)
2. Enter username: `admin`
3. Enter password: `changeme` (or whatever you set in `.env`)
4. Grafana may prompt you to change the password — do so for security

### 15b. Verify the Prometheus Data Source

1. Click the **hamburger menu** (☰) in the top left
2. Go to **Connections** → **Data Sources**
3. You should see **Prometheus** already listed (auto-provisioned)
4. Click it → scroll to the bottom → click **Test**
5. You should see a green **"Successfully queried the Prometheus API"** message

### 15c. View the Pre-loaded Dashboard

1. Click the **hamburger menu** (☰) → **Dashboards**
2. You should see **Node Overview** already listed
3. Click it to open — you'll see gauges for CPU, memory, disk usage, network traffic, and target status

### 15d. Import Community Dashboards (Recommended)

Grafana has thousands of free community dashboards. Here are the best ones for this stack:

| Dashboard | ID | What It Shows |
|---|---|---|
| Node Exporter Full | `1860` | Comprehensive host metrics (CPU, memory, disk, network) |
| Docker Container Monitoring | `893` | Per-container resource usage |
| MySQL Overview | `7362` | MySQL performance, queries, connections |
| Prometheus Stats | `3662` | Prometheus self-monitoring |
| Windows Exporter | `14694` | Windows server metrics (CPU, memory, disk, network) |

**To import a dashboard:**

1. Go to **Dashboards** → **New** → **Import**
2. Enter the dashboard ID (e.g., `1860`)
3. Click **Load**
4. Under "Prometheus", select the **Prometheus** data source from the dropdown
5. Click **Import**
6. The dashboard will appear with live data

---

## Day-to-Day Commands Reference

Run these from the `~/Docker-Project-SYS265` directory.

### Starting and Stopping

```bash
# Start the full stack
docker compose up -d

# Stop all containers (data volumes are preserved)
docker compose down

# Stop and DELETE all data volumes (complete fresh start)
docker compose down -v

# Restart a single service
docker compose restart prometheus
docker compose restart grafana

# Restart the entire stack
docker compose restart
```

### Viewing Logs

```bash
# View logs for a specific service
docker compose logs grafana

# Follow logs in real time
docker compose logs -f prometheus

# Last 100 lines from all services
docker compose logs --tail=100
```

### Checking Status

```bash
# Container status and health
docker compose ps

# Resource usage per container
docker stats --no-stream

# Which images are being used
docker compose images
```

### Updating Images

```bash
# Pull latest versions of all pinned images
docker compose pull

# Recreate containers with new images
docker compose up -d --force-recreate

# Remove old unused images to free disk space
docker image prune -f
```

### Configuration Changes

```bash
# After editing prometheus.yml or alert_rules.yml — hot reload (no downtime):
curl -X POST http://localhost:9090/-/reload

# After editing compose.yaml or .env — recreate affected containers:
docker compose up -d

# After editing alertmanager.yml:
docker compose restart alertmanager

# After editing Grafana provisioning files:
docker compose restart grafana
```

### Shell Access

```bash
# Get a shell inside a container
docker compose exec prometheus sh
docker compose exec grafana bash
docker compose exec nginx-web sh

# Connect to MySQL directly
docker compose exec mysql-db mysql -u root -p
```

---

## Adding More Monitoring Targets

### Monitor Another Linux Server

On the remote server, install Node Exporter:

```bash
# On the remote Ubuntu server:
sudo apt update
sudo apt install -y prometheus-node-exporter
sudo systemctl enable prometheus-node-exporter
sudo systemctl start prometheus-node-exporter
```

Then on your monitoring server, add a new scrape target:

```bash
vim prometheus/prometheus.yml
```

Add under `scrape_configs`:

```yaml
  - job_name: "remote-linux"
    static_configs:
      - targets: ["192.168.1.60:9100"]
        labels:
          instance: "web-server-01"
```

Reload Prometheus (no restart needed):

```bash
curl -X POST http://localhost:9090/-/reload
```

### Monitor a Windows Server

1. Download [windows_exporter](https://github.com/prometheus-community/windows_exporter/releases) on the Windows machine
2. Run the installer — it installs as a Windows service on port 9182
3. Verify: open `http://windows-server-ip:9182/metrics` in a browser

Add to `prometheus/prometheus.yml`:

```yaml
  - job_name: "windows-server"
    static_configs:
      - targets: ["192.168.1.50:9182"]
        labels:
          instance: "win-dc01"
          os: "windows"
```

Reload:

```bash
curl -X POST http://localhost:9090/-/reload
```

---

## Troubleshooting

### Problem: A target shows as DOWN in Prometheus

```bash
# Verify the container is running
docker compose ps

# Check the container's logs
docker compose logs <service-name>

# Test connectivity from the Prometheus container
docker compose exec prometheus wget -qO- http://node-exporter:9100/metrics | head
```

### Problem: Grafana shows "No Data" on dashboards

1. Check the **time range** in the top-right of Grafana — set it to "Last 15 minutes" or "Last 1 hour"
2. Verify Prometheus targets are green at [http://localhost:9090/targets](http://localhost:9090/targets)
3. Test the datasource: Grafana → Connections → Data Sources → Prometheus → **Test**

### Problem: MySQL container keeps restarting

```bash
docker compose logs mysql-db
```

Common causes:
- Incorrect password in `.env` that doesn't match `01-init.sql`
- Port 3306 already in use: `sudo ss -tlnp | grep 3306`
- Insufficient disk space: `df -h`

### Problem: cAdvisor fails to start or shows "mountpoint for cpu not found"

This is a cgroup v2 issue on Ubuntu 22.04+. The `compose.yaml` already includes the fix (`cgroup: host` + `/sys/fs/cgroup` mount), but if you still see errors:

```bash
# Verify cgroup v2
mount | grep cgroup
# Expected: cgroup2 on /sys/fs/cgroup type cgroup2

# Check cAdvisor logs
docker compose logs cadvisor

# Ensure /dev/kmsg is accessible
sudo chmod 644 /dev/kmsg
```

If cAdvisor continues to fail, you can safely comment it out in `compose.yaml` — Node Exporter provides host-level metrics independently. cAdvisor is only needed for per-container metrics.

### Problem: Permission denied on Grafana data

```bash
docker compose exec -u root grafana chown -R grafana:grafana /var/lib/grafana
docker compose restart grafana
```

### Problem: "docker compose" command not found

You likely have the old V1 `docker-compose` (Python-based) instead of V2. Reinstall Docker from the official repository as shown in Step 1. The Compose plugin is included automatically.

```bash
# Check what you have
docker compose version    # V2 (correct)
docker-compose --version  # V1 (outdated)
```

### Problem: Cannot connect from another machine on the network

```bash
# Check your server's IP
hostname -I

# Verify the port is listening
sudo ss -tlnp | grep -E '3000|8080|9090|9093'

# Check UFW isn't blocking
sudo ufw status
```

Use `http://YOUR-SERVER-IP:3000` instead of `http://localhost:3000` when accessing from another machine.

**Important:** This Project Was made with the help of Claude AI as a Tool
