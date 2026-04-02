# Firewalla Grafana Stack

Self-hosted log monitoring for [Firewalla](https://firewalla.com) using Grafana and Loki. Collects Zeek network logs (DNS, connections) and Firewalla ACL alarm logs, then visualizes them through three pre-built dashboards.

## Architecture

```
Firewalla ──syslog/HTTP──▶ Loki (3100) ──────────────────────▶ Grafana (3000)
                            │                                        │
                         log store                             5 dashboards
                        (30-day TTL)

                         blackbox_exporter (9115)
                            ▲         ▲
                 ICMP ping ─┘         └─ HTTP probes
                 (devices)             (services)
                            │
                    Prometheus (9090) ──────────────────────────▶ Grafana (3000)
                         metrics store      ▲
                        (30-day TTL)        │
                                    node_exporter (9100)
                                    ┌───────┴───────┐
                                  pve             pve2
                              (bare-metal)    (bare-metal)
                              CPU/mem/disk    CPU/mem/disk
                              net/fs          net/fs/ZFS
```

Office Display data paths:

```
Firewalla Zeek/ACL ──► Loki ──────────────────────────┐
                                                        ├──► Grafana ──► Office Display
Prometheus ──► blackbox (ICMP/HTTP) ───────────────────┤
           ──► node_exporter (pve) ────────────────────┤
           ──► node_exporter (pve2) ───────────────────┘
```

## Dashboards

| Dashboard | Description |
|-----------|-------------|
| **Network Overview** | Pipeline health stats, log volume over time, top talkers |
| **DNS & Security** | DNS query analysis, NXDOMAIN anomaly detection, blocked connections |
| **Traffic & Devices** | Per-device connection breakdown, protocol mix, bandwidth estimation |
| **Infrastructure Health** | Device reachability (ICMP), service health (HTTP), and latency trends |
| **Office Display** | Kiosk-optimized wall display combining Prometheus metrics and Loki logs |

The DNS & Security and Traffic & Devices dashboards include a **Device IP** filter for drilling into individual hosts.

## Quick Start

### Prerequisites

- Docker and Docker Compose
- Firewalla configured to forward logs to your Loki instance

### Setup

```bash
git clone https://github.com/PitziLabs/firewalla-grafana-stack.git
cd firewalla-grafana-stack

cp .env.example .env
# Edit .env and set a real password
nano .env

docker compose up -d
```

Grafana will be available at `http://<host>:3000` (login: `admin` / your password).

## Configuration

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `GRAFANA_ADMIN_PASSWORD` | `changeme` | Grafana admin password |

### Loki

`loki/loki-config.yml` — Key settings:

- **Retention**: 30 days (`720h`)
- **Storage**: Local filesystem with TSDB index
- **Compaction**: Every 10 minutes with retention cleanup

### Log Labels

Firewalla logs arrive with a `log_source` label:

| Label | Source |
|-------|--------|
| `zeek_dns` | Zeek DNS query logs |
| `zeek_conn` | Zeek connection logs |
| `firewalla_acl` | Firewalla ACL block/alarm logs |

## Node Exporter

node_exporter collects system metrics from your bare-metal Proxmox hosts (CPU, memory, disk I/O, filesystem usage, network throughput, and ZFS ARC stats). It runs as a systemd service on each host — not inside Docker — because it needs direct access to `/proc`, `/sys`, and the ZFS kernel module.

### Deploy

```bash
# Deploy to pve (node 1)
scp scripts/deploy-node-exporter.sh root@<pve-ip>:/tmp/
ssh root@<pve-ip> 'bash /tmp/deploy-node-exporter.sh'

# Deploy to pve2 (node 2)
scp scripts/deploy-node-exporter.sh root@<pve2-ip>:/tmp/
ssh root@<pve2-ip> 'bash /tmp/deploy-node-exporter.sh'
```

The script is idempotent — safe to re-run. It handles download, user creation, systemd unit, and verification.

### Verify

```bash
curl http://<host-ip>:9100/metrics | head
```

### After deployment

Update the `node` job targets in `prometheus/prometheus.yml` with the real host IPs (marked with `# ---- UPDATE THESE IPs ----`), then reload Prometheus:

```bash
docker compose exec prometheus kill -HUP 1
```

> **ZFS note**: ZFS metrics (`node_zfs_*`) are collected automatically when the ZFS kernel module is present. The ZFS ARC Hit Rate panel in the Infrastructure Health dashboard will show "No data" on hosts without ZFS pools — this is expected for pve. Only pve2 has ZFS.

## Home Assistant Integration

Home Assistant exports ~828 entity metrics via its built-in Prometheus integration. Prometheus scrapes these every 60 seconds.

### Setup

1. Enable the Prometheus integration in Home Assistant — add `prometheus:` to your `configuration.yaml` and restart HA
2. Create a long-lived access token in HA: **Profile → Long-Lived Access Tokens → Create Token**
3. Save the token to `prometheus/ha_token`:
   ```bash
   echo "YOUR_TOKEN" > prometheus/ha_token
   ```
4. Restart Prometheus to pick up the new scrape target:
   ```bash
   docker compose restart prometheus
   ```

The office display dashboard shows four smart home panels sourced from HA metrics: lights currently on, Sonos speaker reachability, battery levels, and printer toner levels.

## Project Structure

```
├── docker-compose.yml
├── .env.example
├── scripts/
│   └── deploy-node-exporter.sh # Idempotent node_exporter installer for Proxmox hosts
├── loki/
│   └── loki-config.yml
├── prometheus/
│   ├── prometheus.yml          # Scrape config (ICMP + HTTP probes, HA, node_exporter targets)
│   ├── blackbox.yml            # Blackbox exporter module definitions
│   └── ha_token.example        # Template for HA long-lived access token
└── grafana/
    └── provisioning/
        ├── datasources/
        │   ├── loki.yml
        │   └── prometheus.yml
        ├── playlists/
        │   └── playlists.yml           # Provisioned playlist for office display rotation
        └── dashboards/
            ├── dashboards.yml
            ├── network-overview.json
            ├── dns-security.json
            ├── traffic-devices.json
            ├── infra-health.json
            └── office-display.json     # Kiosk-optimized wall display
```

## Useful Commands

```bash
# Start the stack
docker compose up -d

# View logs
docker compose logs -f loki
docker compose logs -f grafana
docker compose logs -f prometheus
docker compose logs -f blackbox

# Restart after config changes
docker compose restart

# Verify Loki is receiving data
curl -s http://localhost:3100/loki/api/v1/labels | python3 -m json.tool

# Verify Prometheus targets are healthy
curl -s http://localhost:9090/api/v1/targets | python3 -m json.tool

# Manually test a blackbox ICMP probe
curl "http://localhost:9115/probe?target=192.168.1.1&module=icmp"

# Manually test a blackbox HTTP probe
curl "http://localhost:9115/probe?target=http://192.168.1.13:8123&module=http_2xx"

# Stop everything
docker compose down
```

## Kiosk / Wall Display

Anonymous viewer auth is enabled by default (`GF_AUTH_ANONYMOUS_ENABLED=true`, `GF_AUTH_ANONYMOUS_ORG_ROLE=Viewer`). This allows Chromium in kiosk mode to display dashboards without a login session. This stack is LAN-only — do not enable anonymous auth on an internet-exposed Grafana instance.

The kiosk display hardware and OS-level autostart setup is documented in [PitziLabs/homelab-infra](https://github.com/PitziLabs/homelab-infra).

### Kiosk URL

```
http://<host>:3000/d/<dashboard-uid>?kiosk&refresh=30s
```

## Office Display / Kiosk Mode

The **Office Display** dashboard (`firewalla-office-display`) is purpose-built for a wall-mounted screen. It combines Prometheus metrics (ICMP device status, HTTP service health, CPU/RAM gauges, network throughput, ping latency) and Loki log queries (DNS query volume, blocked connections) into a single 1920×1080 layout with no scrolling.

### Kiosk mode URL

```
http://<host>:3000/d/firewalla-office-display?kiosk&refresh=30s
```

Append `&inactive` to hide the kiosk exit controls after 5 seconds of inactivity.

### Playlist rotation

A provisioned playlist ("Office Display Rotation") cycles through all four dashboards every 60 seconds. After restarting Grafana, find the playlist ID at **Dashboards → Playlists**, then open:

```
http://<host>:3000/playlists/play/<playlist-id>?kiosk
```

To create it manually instead: go to **Dashboards → Playlists → New Playlist**, add the four dashboards, and set the interval to 60s.

### Chromium / Raspberry Pi kiosk setup

```bash
chromium-browser --kiosk --app="http://<host>:3000/d/firewalla-office-display?kiosk&refresh=30s"
```

For playlist rotation replace the URL with the playlist play URL above.

### Updating instance IP filters

The CPU and RAM gauge panels filter by IP: `192.168.139.8.*` for pve and `192.168.139.7.*` for pve2. If these differ from your node_exporter targets, update the `instance=~` regex in `grafana/provisioning/dashboards/office-display.json` and restart Grafana.

## License

MIT
