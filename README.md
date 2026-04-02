# Firewalla Grafana Stack

Self-hosted log monitoring for [Firewalla](https://firewalla.com) using Grafana and Loki. Collects Zeek network logs (DNS, connections) and Firewalla ACL alarm logs, then visualizes them through three pre-built dashboards.

## Architecture

```
Firewalla ──syslog/HTTP──▶ Loki (3100) ──────────────────────▶ Grafana (3000)
                            │                                        │
                         log store                             4 dashboards
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

## Dashboards

| Dashboard | Description |
|-----------|-------------|
| **Network Overview** | Pipeline health stats, log volume over time, top talkers |
| **DNS & Security** | DNS query analysis, NXDOMAIN anomaly detection, blocked connections |
| **Traffic & Devices** | Per-device connection breakdown, protocol mix, bandwidth estimation |
| **Infrastructure Health** | Device reachability (ICMP), service health (HTTP), and latency trends |

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

## Project Structure

```
├── docker-compose.yml
├── .env.example
├── scripts/
│   └── deploy-node-exporter.sh # Idempotent node_exporter installer for Proxmox hosts
├── loki/
│   └── loki-config.yml
├── prometheus/
│   ├── prometheus.yml          # Scrape config (ICMP + HTTP probes, node_exporter targets)
│   └── blackbox.yml            # Blackbox exporter module definitions
└── grafana/
    └── provisioning/
        ├── datasources/
        │   ├── loki.yml
        │   └── prometheus.yml
        └── dashboards/
            ├── dashboards.yml
            ├── network-overview.json
            ├── dns-security.json
            ├── traffic-devices.json
            └── infra-health.json
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

## Office Display / Kiosk Mode

The **Infrastructure Health** dashboard is tagged `office-display` and is suitable for a wall monitor. Append `?kiosk&refresh=30s` to the Grafana dashboard URL to hide the navigation bar and auto-refresh every 30 seconds:

```
http://<host>:3000/d/firewalla-infra-health?kiosk&refresh=30s
```

## License

MIT
