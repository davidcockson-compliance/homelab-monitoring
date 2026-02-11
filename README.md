# Home Lab Monitoring Stack

A lightweight monitoring stack for a Proxmox-based home server, providing operational visibility of host infrastructure and Docker containers via Prometheus and Grafana.

## Architecture

```
Proxmox Host (HP ProDesk 600 G3 SFF)
└── Ubuntu VM (Docker + Portainer)
    ├── Prometheus (:9090)      ← scrapes all exporters
    ├── Grafana (:3002)         ← queries Prometheus
    ├── node_exporter (:9100)   ← host CPU/RAM/Disk
    ├── Telegraf (:9273)        ← container metrics via Docker API
    └── [Services: Plex, Gitea, Valheim, Job Radar, Scarlet Helix,
         Homepage, Filebrowser, Syncthing, Immich, Uptime Kuma,
         Nginx Proxy Manager, Web Prototypes, Portainer, ...]
```

Access: All services accessible via Tailscale mesh VPN. No public ports exposed.

## Tech Stack

| Component | Version | Purpose |
|---|---|---|
| Prometheus | v3.8.0 | Metrics collection & storage (14-day retention) |
| Grafana | 12.3.0 | Dashboard visualisation |
| node_exporter | v1.10.2 | Host-level CPU, RAM, Disk, Network metrics |
| Telegraf | 1.33 | Container metrics via Docker socket API |
| Docker Compose | v2 | Service orchestration |

### Why Telegraf instead of cAdvisor?

cAdvisor failed to discover Docker containers due to a non-standard Docker data root (`/mnt/storage/docker` instead of `/var/lib/docker`) combined with cgroup v2 on Ubuntu. After multiple attempts with different cAdvisor versions (v0.51.0, v0.52.1) and configurations, Telegraf with its Docker input plugin was used instead — it reads directly from the Docker socket API and works regardless of the storage driver path.

This is documented as a troubleshooting lesson: always verify your Docker data root (`docker info | grep "Docker Root Dir"`) before choosing a container metrics exporter.

## Quick Start

### Prerequisites

- Docker and Docker Compose installed
- An external Docker network named `monitoring`
- Tailscale (optional, for remote access)

### 1. Create the network

```bash
docker network create monitoring
```

### 2. Set your Grafana password

```bash
export GRAFANA_PASSWORD='your-strong-password-here'
```

### 3. Deploy the stack

```bash
cd ~/monitoring
docker compose up -d
```

### 4. Verify

```bash
# Prometheus healthy
curl http://localhost:9090/-/healthy

# Grafana healthy
curl -s http://localhost:3002/api/health

# node_exporter responding
curl -s http://localhost:9100/metrics | head -5

# Telegraf container metrics
curl -s http://localhost:9273/metrics | grep docker_container_mem_usage | head -3
```

### 5. Connect Grafana to Prometheus

1. Open Grafana at `http://<your-ip>:3002`
2. Log in with `admin` / your password
3. **Connections → Data Sources → Add → Prometheus**
4. URL: `http://prometheus:9090`
5. **Save & Test**

## Dashboard

The "Home Lab Overview" dashboard provides two-tier visibility:

### VM Health Row
Three gauges showing host resource utilisation with traffic-light thresholds:

| Panel | Query | Thresholds |
|---|---|---|
| CPU Usage % | `100 - (avg by(instance_name)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)` | Green <70, Yellow 70-85, Red >85 |
| RAM Usage % | `(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100` | Green <70, Yellow 70-85, Red >85 |
| Disk Usage % | `(1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"}) * 100` | Green <70, Yellow 70-85, Red >85 |

### Container Row
Per-container metrics for all Docker services (monitoring stack filtered out):

| Panel | Type | Query |
|---|---|---|
| Container Memory | Bar gauge | `sort_desc(docker_container_mem_usage{container_name!~"prometheus\|grafana\|telegraf\|node-exporter"})` |
| Container CPU % | Time series | `docker_container_cpu_usage_percent{container_name!~"prometheus\|grafana\|telegraf\|node-exporter"}` |
| Container CPU % | Bar gauge | `sort_desc(docker_container_cpu_usage_percent{container_name!~"prometheus\|grafana\|telegraf\|node-exporter"})` |
| Top 5 Memory | Bar gauge | Top consumers by RAM |
| Container Restarts (24h) | Stat | `changes(docker_container_status{...container_status="running"}[24h]) > 0` |

The dashboard JSON is committed at `grafana/dashboards/home-lab-overview.json`.

## What I Learned

- **Valheim server** is the biggest resource consumer: ~1.1 GiB RAM and 6% CPU even when idle
- **immich_machine_learning** holds 354 MiB in memory for the ML model, even when not processing photos
- **Netdata** at ~300 MiB RAM is heavier than the entire Prometheus + Grafana + Telegraf + node_exporter stack combined (~170 MiB)
- **cAdvisor** doesn't work with non-standard Docker data roots on cgroup v2 — Telegraf is a reliable alternative
- The CPU spike pattern from 12:00-13:30 in the dashboard corresponds to deploying and troubleshooting the monitoring stack itself — a natural baseline event

## What I'd Add Next

| Priority | Enhancement | Why |
|---|---|---|
| 1 | smartctl_exporter | Replace broken Scrutiny with SMART disk health monitoring |
| 2 | pve_exporter | Add Proxmox host-level visibility (requires API token) |
| 3 | Second VM node_exporter | Extend to full infrastructure coverage |
| 4 | Alertmanager | Get notified when things break (e.g. disk >90%) |
| 5 | Splunk + log forwarding | Drill-down from metrics to actual log lines |
| 6 | Cloudflare exporter | Monitor Scarlet Helix project traffic |
| 7 | Grafana provisioning | Auto-load dashboards and datasources from config files |
| 8 | Terraform | Codify the monitoring stack deployment as IaC |

## Repository Structure

```
homelab-monitoring/
├── README.md
├── .env.example
├── .gitignore
├── docker-compose.yml
├── prometheus/
│   └── prometheus.yml
├── telegraf/
│   └── telegraf.conf
├── grafana/
│   └── dashboards/
│       └── home-lab-overview.json
└── docs/
    └── dashboard-screenshot.png
```

## Port Allocation

| Service | Port | Access |
|---|---|---|
| Prometheus | 9090 | Tailscale only |
| Grafana | 3002 | Tailscale only |
| node_exporter | 9100 | Tailscale only |
| Telegraf | 9273 | Tailscale only |
| Docker Engine Metrics | 9323 | Tailscale only |

Ports 3000 and 3001 were already in use by Homepage and Gitea respectively.

## Teardown

```bash
cd ~/monitoring
docker compose down
docker volume rm monitoring_prometheus_data monitoring_grafana_data
docker network rm monitoring
```

## Notes

- Docker data root is `/mnt/storage/docker` (configured in `/etc/docker/daemon.json`)
- Docker engine metrics are enabled via `"metrics-addr": "0.0.0.0:9323"` in daemon.json
- Prometheus scrape interval is 30s (sufficient for home lab; 15s doubles storage)
- Prometheus retention is 14 days
- The `version: "3.8"` key in docker-compose.yml generates a deprecation warning — it's ignored by modern Docker Compose and can be removed

## License

MIT
