# CLAUDE.md

## Project Overview

Firewalla log monitoring stack: Grafana + Loki + Prometheus running in Docker on a Proxmox LXC.
Firewalla pushes Zeek logs (DNS, conn) and ACL alarm logs to Loki via syslog/HTTP.
Prometheus scrapes blackbox_exporter for device reachability (ICMP) and service health (HTTP).
Grafana visualizes everything through five provisioned dashboards, including a kiosk-optimized office display.

## Stack

- **Loki 3.4.2** — log aggregation (port 3100)
- **Grafana OSS 11.5.2** — dashboards (port 3000)
- **Prometheus** — metrics collection (port 9090), 30-day retention
- **blackbox_exporter** — ICMP and HTTP probes (port 9115)
- **node_exporter 1.9.0** — host-level metrics on Proxmox bare-metal nodes (port 9100)
- **Home Assistant Prometheus** — ~828 entity metrics scraped every 60s from HA's built-in `/api/prometheus` endpoint (bearer token auth)
- **Docker Compose** — orchestration (Loki, Grafana, Prometheus, blackbox only)

## File Structure

```
docker-compose.yml                                  # Services: loki, grafana, prometheus, blackbox
loki/loki-config.yml                                # Loki config (TSDB, 30-day retention)
prometheus/prometheus.yml                           # Prometheus scrape config (blackbox relabeling, placeholder IPs)
prometheus/blackbox.yml                             # Blackbox exporter module definitions (icmp, http_2xx)
prometheus/ha_token                                 # HA long-lived access token (gitignored, not committed)
prometheus/ha_token.example                         # Template for the HA token file
grafana/provisioning/datasources/loki.yml           # Auto-provisioned Loki datasource
grafana/provisioning/datasources/prometheus.yml     # Auto-provisioned Prometheus datasource
grafana/provisioning/dashboards/dashboards.yml      # Dashboard provider config
grafana/provisioning/dashboards/*.json              # Dashboard definitions
grafana/provisioning/playlists/playlists.yml        # Provisioned playlist for office display rotation
scripts/deploy-node-exporter.sh                     # Idempotent node_exporter installer for Proxmox bare-metal hosts
```

## Running

```bash
cp .env.example .env   # Set GRAFANA_ADMIN_PASSWORD
docker compose up -d
```

## Key Conventions

- **Dashboard UIDs** follow the pattern `firewalla-<name>` (e.g., `firewalla-network-overview`)
- **Loki labels**: `log_source` is the primary stream selector (`zeek_dns`, `zeek_conn`, `firewalla_acl`)
- **Log parsing**: All Zeek queries use `| json | line_format "{{.log}}" | json` to unwrap the nested JSON
- **Template variables**: DNS & Traffic dashboards use `$device_ip` for per-device filtering
- Dashboard JSON files are Grafana schema v39 — do not add `__inputs` or `__requires` (provisioned, not imported)

## node_exporter Deployment Model

node_exporter runs as a bare-metal **systemd service** on each Proxmox host — not inside Docker. This is required because node_exporter needs direct access to `/proc`, `/sys`, and the ZFS kernel module to collect accurate host-level metrics (CPU, memory, disk I/O, filesystem, network, ZFS ARC).

Deploy with `scripts/deploy-node-exporter.sh`: SCP it to each host and run it over SSH. The script is idempotent — re-running it is safe and will no-op if the correct version is already installed.

Prometheus (running inside Docker) scrapes the bare-metal node_exporter endpoints directly over the host network. The `node` scrape job in `prometheus/prometheus.yml` targets port 9100 on each Proxmox host.

## Prometheus Scrape Config Conventions

- **Blackbox relabeling pattern**: `icmp_ping` and `http_probe` jobs rewrite `__address__` to `blackbox:9115` and preserve the original target as the `instance` label. This is the standard blackbox_exporter pattern.
- **Placeholder IPs**: Target IPs in `prometheus/prometheus.yml` are placeholders — update to match your network. A `# ---- UPDATE THESE IPs ----` comment marks the target blocks.
- **node job**: The `node` job in `prometheus.yml` targets `192.168.1.10:9100` (pve) and `192.168.1.11:9100` (pve2). Update these IPs after deploying node_exporter on your Proxmox hosts.
- **homeassistant job**: Scrapes HA at `192.168.139.172:8123/api/prometheus` every 60s using `bearer_token_file`. Key metric families: `homeassistant_light_brightness_percent`, `homeassistant_sensor_battery_percent`, `homeassistant_entity_available`, `homeassistant_sensor_unit_percent`.

## Loki Query Patterns

Common query structure used across dashboards:
```logql
{log_source="zeek_dns"} | json | line_format "{{.log}}" | json | __error__=""
```

Metric queries use `count_over_time`, `rate`, `sum_over_time` (with `unwrap` for byte fields).

## Anonymous Auth (Kiosk Mode)

Anonymous viewer access is enabled via `GF_AUTH_ANONYMOUS_ENABLED=true` and `GF_AUTH_ANONYMOUS_ORG_ROLE=Viewer` in `docker-compose.yml`. This allows Chromium running in kiosk mode to display dashboards without a login session.

**This is appropriate for a LAN-only instance.** Do not enable anonymous auth on an internet-exposed Grafana deployment — unauthenticated users would have read access to all dashboards and data.

## Office Display Dashboard

`firewalla-office-display` is the kiosk-optimized dashboard for a wall-mounted screen. It mixes both datasources:
- **Prometheus**: Device Status (ICMP), Services (HTTP), CPU gauges, RAM gauges, Network traffic, Ping latency, Lights On, Sonos status (via Home Assistant)
- **Loki**: DNS query volume (`zeek_dns`), Blocked connections (`firewalla_acl`)

Designed for 1920×1080, no scrolling, 26-grid-unit total height (fits within kiosk viewport including Grafana's inter-panel margins and dashboard padding). All panels use `colorMode: "background"` with red/green thresholds for instant readability at a distance.

Kiosk URL:
```
http://<host>:3000/d/firewalla-office-display?kiosk&refresh=30s
```

Playlist rotation URL (after Grafana assigns the playlist an ID):
```
http://<host>:3000/playlists/play/<playlist-id>?kiosk
```

Instance IP filters in CPU/RAM gauge queries match `192.168.139.8.*` (pve) and `192.168.139.7.*` (pve2). Update these if your node_exporter IPs differ.

## Testing Changes

- Validate JSON dashboards: `python3 -m json.tool grafana/provisioning/dashboards/<file>.json > /dev/null`
- Validate YAML configs: `python3 -c "import yaml; yaml.safe_load(open('<file>'))"` (if PyYAML available)
- After editing dashboards, restart Grafana to reload: `docker compose restart grafana`
