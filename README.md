# Firewalla Grafana Stack

Self-hosted log monitoring for [Firewalla](https://firewalla.com) using Grafana and Loki. Collects Zeek network logs (DNS, connections) and Firewalla ACL alarm logs, then visualizes them through three pre-built dashboards.

## Architecture

```
Firewalla ──syslog/HTTP──▶ Loki (3100) ──▶ Grafana (3000)
                            │                  │
                         log store        3 dashboards
                        (30-day TTL)
```

## Dashboards

| Dashboard | Description |
|-----------|-------------|
| **Network Overview** | Pipeline health stats, log volume over time, top talkers |
| **DNS & Security** | DNS query analysis, NXDOMAIN anomaly detection, blocked connections |
| **Traffic & Devices** | Per-device connection breakdown, protocol mix, bandwidth estimation |

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

## Project Structure

```
├── docker-compose.yml
├── .env.example
├── loki/
│   └── loki-config.yml
└── grafana/
    └── provisioning/
        ├── datasources/
        │   └── loki.yml
        └── dashboards/
            ├── dashboards.yml
            ├── network-overview.json
            ├── dns-security.json
            └── traffic-devices.json
```

## Useful Commands

```bash
# Start the stack
docker compose up -d

# View logs
docker compose logs -f loki
docker compose logs -f grafana

# Restart after config changes
docker compose restart

# Verify Loki is receiving data
curl -s http://localhost:3100/loki/api/v1/labels | python3 -m json.tool

# Stop everything
docker compose down
```

## License

MIT
