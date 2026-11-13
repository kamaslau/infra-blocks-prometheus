# infra-blocks-prometheus

[![Blocks](https://img.shields.io/badge/infra--blocks-index-555?style=flat-square)](https://github.com/kamaslau/infra-blocks)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](https://opensource.org/licenses/MIT)
![Repository size](https://img.shields.io/github/repo-size/kamaslau/infra-blocks-prometheus?color=56BEB8)

A unit, not a demo: three services, three ports, one documented credential, disposable with a single command.

Typical use: the first time someone asks how you know the server is still up, and the answer cannot be "I pinged it".

[Prometheus](https://prometheus.io/docs/prometheus/latest/installation/#using-docker) with [Grafana](https://grafana.com/docs/grafana/latest/setup-grafana/installation/docker/) and node-exporter, wired to each other before the first `up`.

## Service URL

- Prometheus `http://localhost:9090` — targets at `/targets`, queries at `/graph`
- Grafana `http://localhost:3000`
- node-exporter `http://localhost:9100/metrics`

Ports can be changed in `.env`; see Usage. Grafana's 3000 is the same default as Gitea in [infra-blocks-gitea](https://github.com/kamaslau/infra-blocks-gitea): when both units run on one machine, change `PORT_GRAFANA` for one of them, or the second container starts with a published port that answers as the other application.

## Default user

Grafana ships with `admin` / `admin` and asks for a new password on first login. Prometheus and node-exporter have no authentication at all — they are scrapers, and exposing either to the internet hands out the host's inventory.

| Service | Username | Password |
| ------- | -------- | -------- |
| Grafana | `admin` | `admin` |

Defaults are for local use only.

## Usage

Open three ports before the first `up`: `3000`, `9090` and `9100` on the monitoring host, and `9100` on every host being monitored.

### The monitoring host

```bash
# Optional: change the host ports (defaults 3000 / 9090 / 9100)
cp .env.example .env

# Prometheus scrapes what prometheus.yml names; the sample scrapes node-exporter
cp -f prometheus.sample.yml prometheus.yml

docker compose up -d
```

`prometheus.yml` is gitignored: the sample is the starting point, and the real one names the targets of a specific site.

### Add the datasource in Grafana

<http://localhost:3000/connections/datasources/new> → Prometheus → `URL` = `http://prometheus:9090` → Save & test.

The hostname is the service name in `compose.yml`, not `localhost:9090`. Grafana runs in the same compose network, so the container name resolves and the published port is irrelevant to it — this is the step where a working stack is usually configured to fail.

### Import a dashboard

<http://localhost:3000/dashboard/import> → `Import via grafana.com` → `1860` → Load → select the `prometheus` datasource → Import.

Dashboard 1860 is the standard node-exporter view: CPU, memory, disk, network, one panel per question a client actually asks.

### Monitored hosts

```bash
docker run --name node-exporter --restart always -d -p 9100:9100 prom/node-exporter
```

Then add the host to `prometheus.yml` under `scrape_configs`, and `docker compose restart prometheus`.

## Verify

```bash
curl -s http://localhost:9090/-/healthy          # Prometheus Server is Healthy.
curl -s http://localhost:3000/api/health         # {"commit":"…","database":"ok","version":"…"}
curl -s http://localhost:9100/metrics | head -3  # node_exporter build info
```

Three answers from three processes. The one that proves the wiring, not just the processes:

```bash
curl -s http://localhost:9090/api/v1/targets | grep -o '"health":"up"' | wc -l
```

A non-zero count means Prometheus reached node-exporter and parsed its metrics. `0` with targets listed means the scrape is configured and failing — the target's `lastError` says whether it is DNS (`no such host`), a refused connection, or a timeout.

## Why this shape

The three services are one unit because none of them is useful alone: Prometheus without a dashboard is a query prompt nobody on the client's side will open, Grafana without a datasource is an empty login page, and node-exporter without a scraper is a metrics endpoint nobody reads.

Grafana reaches Prometheus by service name (`http://prometheus:9090`) on the compose network, and Prometheus scrapes `node-exporter:9100` the same way. Publishing the ports is for the human with a browser; internal traffic never leaves the network. A configuration that uses `localhost:9090` inside Grafana works exactly until the containers are recreated on different hosts.

`prometheus.yml` is bind-mounted from the working directory and gitignored, because it is the one file that differs per site. `prometheus.sample.yml` shows the minimum: one job, one target, a 1m interval — short enough to be useful during a handover, long enough not to fill a small disk.

No Alertmanager, deliberately. Alerting on a single host that nobody has agreed to be on call for produces alerts that are ignored, and ignored alerts are worse than no alerts. When the client has a person who will answer, that is the point to add it — the scrape configuration does not change.

The Grafana volume holds dashboards and the admin password; the Prometheus volume holds the time series, at the image default of 15 days retention. `docker compose down -v` removes both, which is the correct behaviour for a demo and the wrong one for a handover: on a real host, `down` without `-v`.

## Notes for deployments in China

All three images pull from Docker Hub and need a registry mirror. Dashboard 1860 is fetched from `grafana.com` by the browser, and that request can fail independently of the stack — the import dialog shows `Failed to load dashboard` while Grafana itself is healthy. Downloading the JSON once and pasting it into the import box avoids the round trip.

Scraping across the client's firewall needs the `9100` rule to be written per host, and node-exporter has no authentication: publish it on the intranet interface, not on `0.0.0.0`, when the host has a public one.

---

Part of [infrastructure blocks](https://github.com/kamaslau/infra-blocks) · ← back to index
