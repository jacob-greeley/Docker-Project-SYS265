# Prometheus Monitoring Stack

A production-ready, multi-container monitoring infrastructure using Docker Compose on **Ubuntu**. This project deploys **Prometheus**, **Grafana**, **Alertmanager**, **Node Exporter**, **cAdvisor**, **Nginx** (web), **MySQL** (database), and a **MySQL Exporter** — all wired together with customized networking, persistent volumes, health checks, and auto-provisioned dashboards.

> Built for network monitoring in lab environments on Ubuntu 22.04/24.04 LTS (tested on `jacob.local` domain). Includes cgroup v2 compatibility out of the box.

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Quick Start](#quick-start)
- [Step-by-Step Setup Guide](#step-by-step-setup-guide)
  - [Step 1 — Install Docker and Docker Compose](#step-1--install-docker-and-docker-compose)
  - [Step 2 — Clone the Repository](#step-2--clone-the-repository)
  - [Step 3 — Configure Environment Variables](#step-3--configure-environment-variables)
  - [Step 4 — Review and Customize Configuration](#step-4--review-and-customize-configuration)
  - [Step 5 — Deploy the Stack](#step-5--deploy-the-stack)
  - [Step 6 — Verify All Services](#step-6--verify-all-services)
  - [Step 7 — Access the Dashboards](#step-7--access-the-dashboards)
- [Service Details](#service-details)
- [Customization Guide](#customization-guide)
  - [Adding Scrape Targets](#adding-scrape-targets)
  - [Creating Custom Alert Rules](#creating-custom-alert-rules)
  - [Importing Grafana Dashboards](#importing-grafana-dashboards)
- [Docker Compose Explained](#docker-compose-explained)
- [Common Commands](#common-commands)
- [Troubleshooting](#troubleshooting)
- [Security Considerations](#security-considerations)

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                        Docker Host                              │
│                                                                 │
│  ┌──────────────┐   scrapes    ┌──────────────┐                │
│  │  Prometheus   │◄────────────│ Node Exporter │                │
│  │  :9090        │             │ (host metrics)│                │
│  │              │◄──────┐      └──────────────┘                │
│  │              │       │      ┌──────────────┐                │
│  │              │       ├──────│   cAdvisor    │                │
│  │              │       │      │(container met)│                │
│  │              │       │      └──────────────┘                │
│  │              │       │      ┌──────────────┐                │
│  │              │       ├──────│MySQL Exporter │                │
│  │              │       │      │  :9104        │                │
│  └──────┬───────┘       │      └──────┬───────┘                │
│         │               │             │                         │
│         │ alerts        │             │ queries                  │
│         ▼               │             ▼                         │
│  ┌──────────────┐       │      ┌──────────────┐                │
│  │ Alertmanager │       │      │    MySQL      │                │
│  │  :9093        │       │      │  (database)  │                │
│  └──────────────┘       │      └──────────────┘                │
│         │               │                                       │
│         │               │      ┌──────────────┐                │
│         │               └──────│    Nginx      │                │
│         ▼                      │  :8080 (web)  │                │
│  ┌──────────────┐              └──────────────┘                │
│  │   Grafana    │                                               │
│  │  :3000       │                                               │
│  │ (dashboards) │                                               │
│  └──────────────┘                                               │
└─────────────────────────────────────────────────────────────────┘
```

**Networks:**
| Network | Purpose | Services |
|---|---|---|
| `monitoring-net` | Prometheus ↔ exporters ↔ Grafana communication | All monitoring services |
| `backend-net` | MySQL ↔ MySQL Exporter ↔ Prometheus (isolated DB traffic) | mysql, mysql-exporter, prometheus |
| `frontend-net` | Nginx public-facing traffic | nginx |

---

## Prerequisites

| Requirement | Minimum | Recommended |
|---|---|---|
| **OS** | Ubuntu 22.04 LTS (Jammy) | Ubuntu 24.04 LTS (Noble) |
| **Docker Engine** | 24.0+ | 27.0+ |
| **Docker Compose** | v2.20+ (ships with Docker Engine) | v2.30+ |
| **RAM** | 4 GB | 8 GB |
| **Disk** | 10 GB free | 20 GB+ |
| **Ports available** | 3000, 8080, 9090, 9093 | — |

> **Note:** This project uses `docker compose` (Compose V2, no hyphen). The legacy `docker-compose` (V1, Python-based) was deprecated in June 2023. Docker Engine on Ubuntu bundles Compose V2 automatically when installed from the official Docker repository.

> **Ubuntu-specific:** Ubuntu 22.04+ defaults to **cgroup v2**. This project's `compose.yaml` already accounts for this with `cgroupns: host` on the cAdvisor container and the required `/sys/fs/cgroup` mount. No manual cgroup configuration is needed.

---

## Project Structure

```
Docker-Project-SYS265/
├── compose.yaml                    # Main Docker Compose file
├── .env.example                    # Environment variable template
├── .gitignore
├── README.md
├── prometheus/
│   ├── prometheus.yml              # Prometheus scrape configuration
│   └── alert_rules.yml             # Alerting rules (PromQL expressions)
├── grafana/
│   ├── provisioning/
│   │   ├── datasources/
│   │   │   └── prometheus.yml      # Auto-registers Prometheus as data source
│   │   └── dashboards/
│   │       └── dashboards.yml      # Dashboard auto-load configuration
│   └── dashboards/
│       └── node-overview.json      # Pre-built Node Overview dashboard
├── alertmanager/
│   └── alertmanager.yml            # Alert routing and receiver config
├── nginx/
│   ├── conf.d/
│   │   └── default.conf            # Nginx vhost with stub_status
│   └── html/
│       └── index.html              # Landing page with service links
└── mysql/
    └── init/
        └── 01-init.sql             # DB init script + exporter user
```

---

## Quick Start

If you're already familiar with Docker Compose and just want to get running:

```bash
# Download the latest release from GitHub
wget https://github.com/jacob-greeley/Docker-Project-SYS265/releases/latest/download/Prometheus-monitoring-stack.zip

# Unzip into a working directory
mkdir -p ~/Docker-Project-SYS265 && cd ~/Docker-Project-SYS265
unzip ~/Prometheus-monitoring-stack.zip

cp .env.example .env           # Edit passwords in .env
docker compose up -d           # Deploy all services
docker compose ps              # Verify everything is running
```

Then open [http://localhost:3000](http://localhost:3000) (Grafana) and log in with `admin` / `changeme`.

---

## Step-by-Step Setup Guide

### Step 1 — Install Docker and Docker Compose on Ubuntu

Docker Compose V2 is bundled with Docker Engine. Install Docker using the official repository method from Docker's documentation.

> **Important:** Do **not** install Docker from Ubuntu's default `apt` repository (`docker.io` package) — it is outdated and does not include the Compose plugin. Always use Docker's official repository.

**Remove any conflicting packages first:**

```bash
# Remove old or conflicting Docker packages
sudo apt remove docker docker-engine docker.io containerd runc docker-compose 2>/dev/null
```

**Install prerequisites:**

```bash
sudo apt update
sudo apt install -y ca-certificates curl
```

**Add Docker's official GPG key and repository:**

```bash
# Create the keyrings directory
sudo install -m 0755 -d /etc/apt/keyrings

# Download Docker's GPG key
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the Docker repository (DEB822 format — current official method)
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF
```

**Install Docker Engine + Compose plugin:**

```bash
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
```

**Enable Docker to start on boot:**

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

**Run Docker without `sudo` (add your user to the `docker` group):**

```bash
sudo usermod -aG docker $USER
```

> **You must log out and back in** (or run `newgrp docker`) for the group change to take effect.

**Verify installation:**

```bash
docker --version
# Expected: Docker version 27.x.x or higher

docker compose version
# Expected: Docker Compose version v2.30.x or higher

# Quick smoke test
docker run --rm hello-world
```

> **Important:** Use `docker compose` (with a space). The hyphenated `docker-compose` is the legacy V1 binary and is no longer maintained.

**Open required firewall ports (if UFW is enabled):**

```bash
# Check if UFW is active
sudo ufw status

# If active, allow the monitoring stack ports
sudo ufw allow 3000/tcp comment "Grafana"
sudo ufw allow 8080/tcp comment "Nginx web"
sudo ufw allow 9090/tcp comment "Prometheus"
sudo ufw allow 9093/tcp comment "Alertmanager"
sudo ufw reload
```

---

### Step 2 — Download the Project

Download the latest release zip from GitHub:

```bash
wget https://github.com/jacob-greeley/Docker-Project-SYS265/releases/latest/download/Prometheus-monitoring-stack.zip
mkdir -p ~/Docker-Project-SYS265 && cd ~/Docker-Project-SYS265
unzip ~/Prometheus-monitoring-stack.zip
```

Or create the directory manually if building from scratch:

```bash
mkdir -p ~/Docker-Project-SYS265 && cd ~/Docker-Project-SYS265
```

---

### Step 3 — Configure Environment Variables

Copy the example environment file and edit it with your own passwords:

```bash
cp .env.example .env
vim .env
```

The `.env` file is loaded automatically by Docker Compose and contains:

```env
# Grafana
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=changeme      # ← CHANGE THIS

# MySQL
MYSQL_ROOT_PASSWORD=rootpassword     # ← CHANGE THIS
MYSQL_DATABASE=monitoring_db
MYSQL_USER=appuser
MYSQL_PASSWORD=apppassword           # ← CHANGE THIS
MYSQL_EXPORTER_PASSWORD=exporterpass # ← CHANGE THIS
```

> **Security:** The `.env` file contains secrets and is excluded from git via `.gitignore`. Never commit it to version control.

---

### Step 4 — Review and Customize Configuration

#### 4a. Prometheus Scrape Config (`prometheus/prometheus.yml`)

This file tells Prometheus what to monitor. By default it scrapes itself, Node Exporter, cAdvisor, MySQL Exporter, and Nginx.

Key settings:

```yaml
global:
  scrape_interval: 15s       # How frequently to pull metrics
  evaluation_interval: 15s   # How frequently to evaluate alert rules
```

Each `scrape_configs` entry defines a target. Targets use **container names** as hostnames because they share the `monitoring-net` Docker network:

```yaml
scrape_configs:
  - job_name: "node-exporter"
    static_configs:
      - targets: ["node-exporter:9100"]
```

#### 4b. Alert Rules (`prometheus/alert_rules.yml`)

Pre-configured alerts include host down, high CPU (>85%), high memory (>85%), low disk space (<15%), container resource alerts, MySQL down, and MySQL connection saturation. All thresholds are tunable by editing the PromQL expressions.

#### 4c. Alertmanager (`alertmanager/alertmanager.yml`)

By default, alerts are received but not forwarded anywhere. To enable notifications, uncomment and configure the `webhook_configs`, `email_configs`, or `slack_configs` section in this file.

#### 4d. Grafana Provisioning

The `grafana/provisioning/` directory auto-configures Grafana on first boot so that Prometheus is registered as a data source and the included dashboard JSON files are loaded automatically. No manual setup is required.

---

### Step 5 — Deploy the Stack

```bash
# Pull all images first (optional, shows download progress cleanly)
docker compose pull

# Start all services in detached mode
docker compose up -d
```

Expected output:

```
[+] Running 9/9
 ✔ Network monitoring-net   Created
 ✔ Network frontend-net     Created
 ✔ Network backend-net      Created
 ✔ Volume "prometheus-data" Created
 ✔ Volume "grafana-data"    Created
 ✔ Volume "mysql-data"      Created
 ✔ Container prometheus     Started
 ✔ Container mysql-db       Started
 ✔ Container node-exporter  Started
 ✔ Container cadvisor       Started
 ✔ Container alertmanager   Started
 ✔ Container mysql-exporter Started
 ✔ Container grafana        Started
 ✔ Container nginx-web      Started
```

---

### Step 6 — Verify All Services

```bash
# Check container status (all should show "Up" or "Up (healthy)")
docker compose ps
```

Expected output:

```
NAME              IMAGE                           STATUS                   PORTS
alertmanager      prom/alertmanager:v0.27.0       Up (healthy)             0.0.0.0:9093->9093/tcp
cadvisor          gcr.io/cadvisor/cadvisor:v0.49.1 Up                      8080/tcp
grafana           grafana/grafana:11.4.0          Up (healthy)             0.0.0.0:3000->3000/tcp
mysql-db          mysql:8.0                       Up (healthy)             3306/tcp
mysql-exporter    prom/mysqld-exporter:v0.16.0    Up                       9104/tcp
nginx-web         nginx:1.27-alpine               Up (healthy)             0.0.0.0:8080->80/tcp
node-exporter     prom/node-exporter:v1.8.2       Up                       9100/tcp
prometheus        prom/prometheus:v2.53.5          Up (healthy)             0.0.0.0:9090->9090/tcp
```

**Verify Prometheus targets are being scraped:**

```bash
# Check Prometheus targets API
curl -s http://localhost:9090/api/v1/targets | python3 -m json.tool | head -30
```

Or open [http://localhost:9090/targets](http://localhost:9090/targets) in a browser — all targets should show **UP** (green).

**Check logs if any container is unhealthy:**

```bash
docker compose logs prometheus    # Check specific service
docker compose logs --tail=50     # Last 50 lines from all services
```

---

### Step 7 — Access the Dashboards

| Service | URL | Credentials |
|---|---|---|
| **Grafana** | [http://localhost:3000](http://localhost:3000) | `admin` / `changeme` (or your `.env` values) |
| **Prometheus** | [http://localhost:9090](http://localhost:9090) | No auth |
| **Alertmanager** | [http://localhost:9093](http://localhost:9093) | No auth |
| **Nginx (web)** | [http://localhost:8080](http://localhost:8080) | No auth |

**First time in Grafana:**

1. Log in with the admin credentials
2. The **Prometheus data source** is already configured (auto-provisioned)
3. Go to **Dashboards** → the **Node Overview** dashboard is pre-loaded
4. For richer dashboards, import community dashboards by ID (see [Importing Grafana Dashboards](#importing-grafana-dashboards))

---

## Service Details

| Service | Image | Purpose | Port |
|---|---|---|---|
| `prometheus` | `prom/prometheus:v2.53.5` | Time-series database, scrapes and stores metrics | 9090 |
| `grafana` | `grafana/grafana:11.4.0` | Dashboard visualization and alerting UI | 3000 |
| `alertmanager` | `prom/alertmanager:v0.27.0` | Routes alerts to receivers (email, Slack, webhooks) | 9093 |
| `node-exporter` | `prom/node-exporter:v1.8.2` | Exports host-level metrics (CPU, memory, disk, network) | 9100 (internal) |
| `cadvisor` | `gcr.io/cadvisor/cadvisor:v0.49.1` | Exports per-container resource metrics | 8080 (internal) |
| `mysql-exporter` | `prom/mysqld-exporter:v0.16.0` | Exports MySQL server metrics | 9104 (internal) |
| `nginx` | `nginx:1.27-alpine` | Web server with `stub_status` metrics endpoint | 8080 |
| `mysql` | `mysql:8.0` | Relational database | 3306 (internal) |

---

## Customization Guide

### Adding Scrape Targets

To monitor additional hosts or services on your network, add entries to `prometheus/prometheus.yml`:

**Example — monitor a Windows server running `windows_exporter`:**

```yaml
scrape_configs:
  # ... existing jobs ...

  - job_name: "windows-server"
    static_configs:
      - targets: ["192.168.1.50:9182"]
        labels:
          instance: "win-dc01"
          os: "windows"
```

**Example — monitor another Linux server running Node Exporter:**

```yaml
  - job_name: "remote-linux"
    static_configs:
      - targets: ["192.168.1.60:9100"]
        labels:
          instance: "web-server-01"
```

After editing, reload Prometheus without restarting:

```bash
# Hot reload (requires --web.enable-lifecycle flag, which is set in compose.yaml)
curl -X POST http://localhost:9090/-/reload
```

---

### Creating Custom Alert Rules

Add new alert groups to `prometheus/alert_rules.yml`. The format is:

```yaml
groups:
  - name: my-custom-alerts
    rules:
      - alert: AlertName
        expr: <PromQL expression that returns results when alert should fire>
        for: 5m          # How long the condition must be true before firing
        labels:
          severity: warning    # or critical
        annotations:
          summary: "Short description"
          description: "Detailed description with {{ $labels.instance }} and {{ $value }}"
```

After editing, reload Prometheus:

```bash
curl -X POST http://localhost:9090/-/reload
```

Verify rules loaded at [http://localhost:9090/rules](http://localhost:9090/rules).

---

### Importing Grafana Dashboards

Grafana has thousands of community dashboards at [https://grafana.com/grafana/dashboards/](https://grafana.com/grafana/dashboards/).

**Recommended dashboard IDs to import:**

| Dashboard | ID | Use Case |
|---|---|---|
| Node Exporter Full | `1860` | Comprehensive host metrics |
| Docker Container Monitoring | `893` | Per-container resource usage |
| MySQL Overview | `7362` | MySQL performance and queries |
| Prometheus Stats | `3662` | Prometheus self-monitoring |

**To import via the Grafana UI:**

1. Go to **Dashboards** → **New** → **Import**
2. Enter the dashboard ID (e.g., `1860`)
3. Click **Load**
4. Select **Prometheus** as the data source
5. Click **Import**

**To import via file (auto-provisioning):**

1. Download the dashboard JSON from grafana.com
2. Save it to `grafana/dashboards/`
3. Restart Grafana: `docker compose restart grafana`

---

## Docker Compose Explained

Key design decisions in the `compose.yaml`:

**No `version` key:** The top-level `version` property (e.g., `version: '3.8'`) is obsolete in Docker Compose V2+ and triggers a warning. Modern Compose automatically uses the latest schema. This project omits it intentionally.

**Ubuntu cgroup v2 compatibility:** Ubuntu 22.04+ defaults to cgroup v2 (unified hierarchy), which can cause issues with container metric collectors. The cAdvisor service includes `cgroupns: host` and mounts `/sys/fs/cgroup:ro` to ensure it can read cgroup data correctly under the v2 hierarchy. The `--docker_only=true` flag limits cAdvisor to Docker containers only, and `--disable_metrics` disables metric collectors known to be problematic on cgroup v2.

**Named volumes for persistence:** All stateful services (Prometheus, Grafana, MySQL, Alertmanager) use named Docker volumes. Data survives container restarts, image upgrades, and `docker compose down`. To destroy data, you must explicitly pass the `-v` flag.

**Three isolated networks:** Services are segmented into `monitoring-net`, `backend-net`, and `frontend-net`. MySQL is not exposed to the monitoring network directly — only through the exporter. Nginx sits on both the monitoring and frontend networks.

**Health checks on critical services:** Prometheus, Grafana, Nginx, and MySQL have health checks defined. Dependent services (like `grafana` depending on `prometheus`) use `condition: service_healthy` to ensure proper startup order.

**Read-only volume mounts (`:ro`):** Configuration files are mounted read-only to prevent containers from modifying the host files.

**Environment variable substitution:** Sensitive values like passwords use `${VAR:-default}` syntax, pulling from the `.env` file at runtime.

---

## Common Commands

```bash
# Start the full stack
docker compose up -d

# Stop all containers (preserves data volumes)
docker compose down

# Stop and DELETE all data volumes (fresh start)
docker compose down -v

# Restart a single service
docker compose restart prometheus

# View logs (follow mode)
docker compose logs -f grafana

# View logs for all services (last 100 lines)
docker compose logs --tail=100

# Check running containers and health status
docker compose ps

# Pull latest images (for version-pinned tags this is a no-op)
docker compose pull

# Rebuild and restart after config changes
docker compose up -d --force-recreate

# Hot-reload Prometheus config (no downtime)
curl -X POST http://localhost:9090/-/reload

# Execute a shell inside a container
docker compose exec prometheus sh
docker compose exec mysql-db mysql -u root -p

# Check resource usage per container
docker stats
```

---

## Troubleshooting

**Problem: A target shows as DOWN in Prometheus**

```bash
# Verify the container is running
docker compose ps <service-name>

# Check the container logs
docker compose logs <service-name>

# Test connectivity from the Prometheus container
docker compose exec prometheus wget -qO- http://node-exporter:9100/metrics | head
```

**Problem: Grafana shows "No Data" on dashboards**

Possible causes and fixes:
- Confirm the time range in Grafana is set to "Last 1 hour" or "Last 15 minutes" (not a future date)
- Check that Prometheus is scraping targets at [http://localhost:9090/targets](http://localhost:9090/targets)
- Verify the datasource is configured: Grafana → Connections → Data Sources → Prometheus → Test

**Problem: MySQL container keeps restarting**

```bash
docker compose logs mysql-db
```

Common causes: incorrect password in `.env`, port 3306 already in use on the host, or insufficient disk space for initialization.

**Problem: cAdvisor fails to start or shows "mountpoint for cpu not found"**

This is a cgroup v2 issue common on Ubuntu 22.04+. The `compose.yaml` already includes the fix (`cgroupns: host` + `/sys/fs/cgroup` mount), but if you still see errors:

```bash
# Verify your system uses cgroup v2
mount | grep cgroup
# Expected: cgroup2 on /sys/fs/cgroup type cgroup2

# Check cAdvisor logs for specific errors
docker compose logs cadvisor

# Ensure /dev/kmsg is accessible
sudo chmod 644 /dev/kmsg
```

If cAdvisor continues to fail, you can safely comment it out in `compose.yaml` — Node Exporter provides host-level metrics independently. cAdvisor is only needed for per-container metrics.

**Problem: Permission denied on Grafana data**

```bash
# Fix ownership for the Grafana named volume
docker compose exec -u root grafana chown -R grafana:grafana /var/lib/grafana
docker compose restart grafana
```

---

## Security Considerations

This stack is designed for lab and internal network use. For production environments, consider the following hardening steps:

1. **Change all default passwords** in `.env` before first deployment
2. **Do not expose ports to the public internet** — use a reverse proxy (Traefik, Nginx) with TLS
3. **Configure UFW firewall rules** — Ubuntu ships with UFW; restrict access to monitoring ports to your local network only:
   ```bash
   # Allow only your lab subnet (example: 192.168.1.0/24)
   sudo ufw allow from 192.168.1.0/24 to any port 3000 proto tcp comment "Grafana - LAN only"
   sudo ufw allow from 192.168.1.0/24 to any port 9090 proto tcp comment "Prometheus - LAN only"
   sudo ufw allow from 192.168.1.0/24 to any port 9093 proto tcp comment "Alertmanager - LAN only"
   sudo ufw deny 3000/tcp
   sudo ufw deny 9090/tcp
   sudo ufw deny 9093/tcp
   ```
4. **Enable Grafana HTTPS** via `GF_SERVER_PROTOCOL=https` and providing certificates
5. **Restrict Prometheus and Alertmanager access** — these services have no built-in authentication; use a reverse proxy with basic auth or OAuth
6. **Use Docker secrets** instead of environment variables for passwords in Swarm deployments
7. **Keep images updated** — periodically run `docker compose pull` and redeploy
8. **Limit container capabilities** — remove the `privileged` flag from cAdvisor if your kernel allows it
9. **Docker and UFW caveat** — Docker manipulates iptables directly and can bypass UFW rules for published ports. For strict network control, consider using `ports: "127.0.0.1:9090:9090"` syntax to bind services to localhost only, then access them through an authenticated reverse proxy

---

## License

This project is provided as-is for educational and lab use. All Docker images are subject to their respective licenses (Apache 2.0 for Prometheus components, AGPL for Grafana).
