# CPU / Memory / GPU Monitoring & Alerting

Stack: Prometheus + Grafana + node-exporter + NVIDIA DCGM Exporter + Alertmanager, all managed via Docker Compose.

## Directory Structure

```
monitoring/
├── compose.yaml                        # Defines all five services (prometheus, alertmanager, node-exporter, dcgm-exporter, grafana)
├── prometheus/
│   ├── prometheus.yml                  # Scrape targets, alerting, and rule_files config
│   └── rules/
│       └── alerts.yml                  # Alert rules (disk space, GPU temp, XID errors, target down)
├── alertmanager/
│   ├── alertmanager.yml.example        # Config template — copy to alertmanager.yml and fill in your own SMTP info
│   └── alertmanager.yml                # Real credentials, excluded via .gitignore, never committed
└── grafana/
    └── provisioning/
        ├── datasources/datasource.yml  # Registers the Prometheus data source with a pinned uid: prometheus
        └── dashboards/
            ├── dashboards.yml          # Dashboard provisioning config
            └── json/                   # Dashboard definitions, auto-loaded by Grafana on startup
```

## Access

| Service                   | URL                           | Credentials                                        |
| ------------------------- | ----------------------------- | -------------------------------------------------- |
| Grafana                   | http://localhost:3000         | admin/admin on first login — change it immediately |
| Prometheus                | http://localhost:9090         | none                                               |
| node-exporter raw metrics | http://localhost:9100/metrics | none                                               |
| dcgm-exporter raw metrics | http://localhost:9400/metrics | none                                               |
| Alertmanager              | http://localhost:9093         | none                                               |

Bundled Grafana dashboards (auto-provisioned from `grafana/provisioning/dashboards/json/`, no manual import needed):

- **Node Exporter Full** (originally grafana.com ID 1860) — CPU / memory / disk / network
- **NVIDIA DCGM Exporter Dashboard** (originally grafana.com ID 12239, official NVIDIA template) — GPU utilization / VRAM / temperature / power / Tensor Core utilization

## Alert Rules

Defined in `prometheus/rules/alerts.yml`, delivered as email via Alertmanager (recipient configured in `alertmanager/alertmanager.yml` — copy `alertmanager.yml.example` and fill in your own email and SMTP details):

| Alert              | Condition                                                                | Severity |
| ------------------ | ------------------------------------------------------------------------ | -------- |
| TargetDown         | Any Prometheus target down for more than 2 min                           | critical |
| DiskSpaceLow       | Root partition free space < 10% for 5 min                                | critical |
| GPUHighTemperature | GPU temperature > 85°C for 5 min                                         | warning  |
| GPUXidError        | Any XID error code appears (may indicate a dropped GPU / hardware fault) | critical |

Routing policy (`alertmanager/alertmanager.yml`): notifications for the same alert name are grouped within a 30s window (`group_wait`); unresolved alerts are re-sent every 3 hours (`repeat_interval`).

### Running Without Email Notifications

`alertmanager.yml` still has to exist as a file for the bind mount in `compose.yaml` to work — if it's missing, Docker creates an empty directory in its place and the container fails to start. If you don't want email alerts, skip the SMTP setup and use a null receiver instead:

```yaml
route:
  receiver: "null"
  group_by: ["alertname"]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 3h

receivers:
  - name: "null"
```

Alertmanager still starts normally, and rules keep evaluating and showing up under Prometheus's `/alerts` page and Alertmanager's own UI (`:9093`) — it just drops notifications at the final send step instead of emailing them. Switch back to the `email` receiver in `alertmanager.yml.example` any time you want notifications turned on.

## Prerequisites

- Docker + Docker Compose plugin
- NVIDIA driver + `nvidia-container-toolkit`
- `nvidia-persistenced` service active (`systemctl is-active nvidia-persistenced`)
- If `/etc/nvidia-container-runtime/config.toml` has `mode = "cdi"`, make sure the CDI spec is up to date (see [TROUBLESHOOTING.md](TROUBLESHOOTING.md) if you hit issues)

## Start From Scratch

```bash
cp alertmanager/alertmanager.yml.example alertmanager/alertmanager.yml
# Edit alertmanager/alertmanager.yml — fill in your SMTP details and recipient email
docker compose up -d
docker compose ps -a   # confirm all five containers (prometheus/grafana/node-exporter/dcgm-exporter/alertmanager) are Up
```

## Verification

```bash
curl http://localhost:9090/-/healthy
curl http://localhost:3000/api/health
curl -s http://localhost:9090/api/v1/targets | python3 -c "
import json,sys
d=json.load(sys.stdin)
for t in d['data']['activeTargets']:
    print(t['labels']['job'], t['health'])
"
# expect all three targets (prometheus / node-exporter / dcgm-exporter) to be up
```

After editing `prometheus/prometheus.yml`, the Prometheus container won't pick up the change automatically (bind-mounted file contents aren't part of Compose's rebuild hash) — you need to `docker restart prometheus` manually. Adding or changing a volume mount requires `docker compose up -d <service>` to recreate the container; `restart` alone isn't enough (see [TROUBLESHOOTING.md](TROUBLESHOOTING.md#rule_files-mount-added-but-the-rule-list-is-still-empty)).

## Adding a Custom Dashboard

The two bundled dashboards are provisioned automatically from `grafana/provisioning/dashboards/json/` — nothing to do there. To pull in an additional dashboard from grafana.com (or update one of the bundled ones to a newer revision), fetch it through Grafana's API and drop the result straight into that folder as a `.json` file; the file provisioner picks up new files within `updateIntervalSeconds` (30s), no restart needed:

```bash
# 1. Look up the Prometheus data source's uid
curl -s -u admin:<password> http://localhost:3000/api/datasources

# 2. Fetch the dashboard json from grafana.com through Grafana's gnet proxy, rewrite the
#    datasource reference to the pinned uid, and save it into the provisioning folder
python3 - <<'EOF'
import urllib.request, json, base64

DASH_ID = 1860  # replace with the grafana.com dashboard id you want
DATASOURCE_UID = "prometheus"

auth = base64.b64encode(b"admin:<password>").decode()
headers = {"Authorization": f"Basic {auth}"}
req = urllib.request.Request(f"http://localhost:3000/api/gnet/dashboards/{DASH_ID}", headers=headers)
gnet = json.loads(urllib.request.urlopen(req).read())

dash = gnet["json"]
dash["id"] = None
raw = json.dumps(dash, indent=2).replace("${DS_PROMETHEUS}", DATASOURCE_UID)

with open(f"grafana/provisioning/dashboards/json/{gnet['slug']}.json", "w") as f:
    f.write(raw)
print(f"saved grafana/provisioning/dashboards/json/{gnet['slug']}.json")
EOF
```

Some community dashboards reference the datasource via a `${ds_prometheus}`-style template variable instead of a literal uid (Node Exporter Full does this) — those work out of the box with any single Prometheus datasource and don't need the string replacement above. Check the dashboard's `templating.list` for an entry with `"type": "datasource"` to tell which kind you're dealing with.

## Common Operations

```bash
docker compose logs -f prometheus     # tail a service's logs
docker compose restart dcgm-exporter
docker restart prometheus             # required after editing prometheus.yml
docker compose down                   # stop and remove containers (volumes kept)
docker compose down -v                # also remove volumes (historical data is lost)
```

## Possible Future Improvements

- Prometheus retains data for 15 days by default; add `--storage.tsdb.retention.time=90d` (or similar) to the prometheus command in `compose.yaml` for longer retention
- Current alert thresholds (85°C for GPU, 10% for disk) are rough starting points — tune `prometheus/rules/alerts.yml` after observing real workload behavior
- If email notifications aren't timely enough (e.g. you're asleep), add a Slack or other webhook-based receiver to the `receivers` in `alertmanager.yml` — the same alert can fan out to multiple channels

## Troubleshooting

See [TROUBLESHOOTING.md](TROUBLESHOOTING.md) for issues encountered while setting this up and how they were resolved.
