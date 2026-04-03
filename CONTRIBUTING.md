# Contributing

This is a personal project running a self-hosted Grafana + Loki + Prometheus
observability stack for a Firewalla home network. It runs in my home lab on a
Proxmox LXC container and doubles as a portfolio piece — so stability and
clarity matter.

## Reporting Issues

Issues are welcome! If you've tried running this stack and hit a problem,
please open an issue with:

- What you expected to happen
- What actually happened
- Your host OS and Docker/Docker Compose versions
- Any relevant log output (`docker compose logs <service>`)

## Suggesting Features

Feature ideas are appreciated. Open an issue describing the use case — *why*
you want it matters more than *how* you'd build it.

## Pull Requests

This repo reflects my own infrastructure, so I'm selective about PRs. Before
investing time in a contribution:

1. **Open an issue first** to discuss the change
2. **Keep scope small** — one fix or feature per PR
3. **Follow existing conventions** — dashboard UID pattern (`firewalla-<n>`),
   Grafana provisioning format (schema v39, no `__inputs`/`__requires`),
   Loki label patterns, Docker Compose service naming
4. **Don't add Docker services** without discussion — each service adds
   resource overhead to the LXC
5. **Keep dashboards provisioned** — all dashboard JSON is auto-loaded from
   `grafana/provisioning/dashboards/`, not manually imported

I'll review PRs personally and may adapt contributions to fit the stack's
architecture.

## Code of Conduct

Be kind, be constructive, be respectful. Life's too short for anything else.
