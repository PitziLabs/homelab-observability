# CLAUDE.md

## Project Overview

Firewalla log monitoring stack: Grafana + Loki running in Docker on a Proxmox LXC.
Firewalla pushes Zeek logs (DNS, conn) and ACL alarm logs to Loki via syslog/HTTP.
Grafana visualizes everything through three provisioned dashboards.

## Stack

- **Loki 3.4.2** — log aggregation (port 3100)
- **Grafana OSS 11.5.2** — dashboards (port 3000)
- **Docker Compose** — orchestration

## File Structure

```
docker-compose.yml                           # Services: loki, grafana
loki/loki-config.yml                         # Loki config (TSDB, 30-day retention)
grafana/provisioning/datasources/loki.yml    # Auto-provisioned Loki datasource
grafana/provisioning/dashboards/dashboards.yml  # Dashboard provider config
grafana/provisioning/dashboards/*.json       # Dashboard definitions
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

## Loki Query Patterns

Common query structure used across dashboards:
```logql
{log_source="zeek_dns"} | json | line_format "{{.log}}" | json | __error__=""
```

Metric queries use `count_over_time`, `rate`, `sum_over_time` (with `unwrap` for byte fields).

## Testing Changes

- Validate JSON dashboards: `python3 -m json.tool grafana/provisioning/dashboards/<file>.json > /dev/null`
- Validate YAML configs: `python3 -c "import yaml; yaml.safe_load(open('<file>'))"` (if PyYAML available)
- After editing dashboards, restart Grafana to reload: `docker compose restart grafana`
