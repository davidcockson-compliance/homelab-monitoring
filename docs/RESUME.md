# Resume Guide — Monitoring Stack

## What's Done ✅

- Phase A: Prometheus + Grafana deployed and connected
- Phase B: node_exporter running, VM CPU/RAM/Disk metrics flowing
- Phase C: Telegraf running, all 23 Docker containers visible by name
- Phase D: "Home Lab Overview" dashboard built with VM Health gauges + Container panels
- Docker engine metrics enabled (`/etc/docker/daemon.json`)

## What's Running on the Server

| Container | Port | Status |
|---|---|---|
| prometheus | 9090 | Running |
| grafana | 3002 | Running |
| node-exporter | 9100 | Running |
| telegraf | 9273 | Running |

Dashboard accessible at: `http://100.97.203.77:3002`

## What Still Needs Doing

### Phase E — Documentation & GitHub Push

1. **Export dashboard JSON** from Grafana:
   - Dashboard → Settings → JSON Model → Ctrl+A → Ctrl+C
   - Save to `~/monitoring/grafana/dashboards/home-lab-overview.json`

2. **Save dashboard screenshot** to `~/monitoring/docs/dashboard-screenshot.png`
   - Use SCP from desktop: `scp screenshot.png dave@100.97.203.77:~/monitoring/docs/`
   - Or upload via Filebrowser

3. **Copy clean config files** from this document to the server:
   - The files in this repo are the clean versions
   - Your server files may have the `version: "3.8"` line and leftover cadvisor config
   - Compare and clean up if needed

4. **Init git and push**:
   ```bash
   cd ~/monitoring
   git init
   git add .
   git commit -m "feat: MVP monitoring stack - Prometheus, Grafana, node_exporter, Telegraf"
   git remote add origin git@github.com:davidcockson-compliance/homelab-monitoring.git
   git push -u origin main
   ```

### Cleanup Tasks

- Remove dead cadvisor container: `docker stop cadvisor 2>/dev/null; docker rm cadvisor 2>/dev/null`
- Remove orphan containers: `docker compose up -d --remove-orphans`
- Remove `version: "3.8"` from docker-compose.yml (generates deprecation warning)
- Remove any leftover cadvisor/docker-exporter service blocks from docker-compose.yml
- Consider removing Netdata after verifying Prometheus covers the same metrics (saves ~300 MiB RAM)

### Known Issues

- **Disk at 89%** — investigate what's consuming space in `/mnt/storage/docker`
  ```bash
  sudo du -sh /mnt/storage/docker/* | sort -rh | head -10
  ```
- **docker-exporter container** may still exist as orphan — remove it
- **Telegraf Docker group ID** is hardcoded to `988` — if Docker group changes, update `user` in docker-compose.yml
  ```bash
  getent group docker  # verify group ID
  ```

### Nice-to-Have Polish

- Fix Container Restarts panel "No data" display — edit panel → No data option → show "0"
- Add panel descriptions explaining what each metric means
- Set dashboard as Grafana home dashboard
