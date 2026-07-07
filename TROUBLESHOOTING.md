# Troubleshooting Log

Issues actually encountered while setting up the stack in [README.md](README.md), recorded in the order they came up, for reference when reproducing the setup or debugging similar problems.

## Port conflicts (9090 / 3000 already in use)

This machine is a shared server — 9090 (Prometheus's default port) was already bound by `mihomo` (a local proxy client's control API, which happens to default to port 9090 too), and 3000 (Grafana's default port) was taken by another process at the time.
`compose.yaml` itself keeps the defaults (9090/3000) so the repo stays portable. If your own machine has a conflict, don't edit `compose.yaml` directly — add a git-ignored `compose.override.yaml` next to it with just the port remap, e.g.:

```yaml
services:
  prometheus:
    ports: !override
      - "9091:9090"
  grafana:
    ports: !override
      - "3001:3000"
```

Docker Compose merges `compose.override.yaml` on top of `compose.yaml` automatically — no extra flags needed. The `!override` tag matters here: by default Compose _appends_ list fields like `ports` instead of replacing them, so without it Prometheus would end up publishing both 9090 and 9091 and still fail to bind the already-taken 9090. `.gitignore` excludes `*.override.yaml`/`*.override.yml` so this stays host-local. The container-internal ports are unchanged either way.

## GPU exporter fails to start: missing nvidia-persistenced socket

```
open /run/nvidia-persistenced/socket: no such file or directory
```

Newer `nvidia-container-toolkit` versions (CDI mode) mount the `nvidia-persistenced` socket into the container by default. If the host isn't running that service, this fails.
Fix:

```bash
sudo systemctl start nvidia-persistenced
systemctl is-active nvidia-persistenced   # should report active
```

## GPU exporter fails to start: stale CDI spec references a missing library

```
open /usr/lib/.../libnvidia-egl-wayland.so.1.1.21: no such file or directory
```

The CDI spec files (`/etc/cdi/nvidia.yaml` and `/var/run/cdi/nvidia.yaml`) were stale, referencing a driver library filename that no longer existed (the driver package renamed/rebumped the library but the CDI spec was never regenerated). **Both paths need regenerating**, since `nvidia-container-runtime`'s `spec-dirs` searches both directories:

```bash
sudo nvidia-ctk cdi generate --output=/etc/cdi/nvidia.yaml
sudo nvidia-ctk cdi generate --output=/var/run/cdi/nvidia.yaml
```

## GPU exporter panics on invalid metric names

```
... is not a valid metric name
```

Early on, `utkuozdemir/nvidia_gpu_exporter:1.2.1` was too old to recognize the new unit-suffixed field names (e.g. `[us]`, `[W/s]`) that newer drivers (Blackwell generation) added to `nvidia-smi -q -x` output, generating invalid Prometheus metric names and panicking outright. The fix at the time was switching to the `:latest` tag; the stack has since moved to dcgm-exporter entirely, so this no longer applies.

## Prometheus scrape shows a transient DNS error

```
dial tcp: lookup ... server misbehaving
```

Right after a container is recreated, Docker's embedded DNS (127.0.0.11) can occasionally return a transient resolution failure. It's self-healing — the next scrape (15s interval) recovers on its own, no action needed.

## Switching from nvidia_gpu_exporter to dcgm-exporter

`nvidia_gpu_exporter` shells out to `nvidia-smi` and parses its text output, so the available metrics are limited. `dcgm-exporter` is NVIDIA's official exporter, reading driver counters directly via the DCGM library — richer metrics (Tensor Core, NVLink, ECC, XID errors, etc.) and lower overhead. Migration steps:

1. In `compose.yaml`, replace the `nvidia-gpu-exporter` service with the official image `nvcr.io/nvidia/k8s/dcgm-exporter:<tag>` (find a current tag on [NGC](https://catalog.ngc.nvidia.com/orgs/nvidia/teams/k8s/containers/dcgm-exporter)), keep `runtime: nvidia`, add `cap_add: [SYS_ADMIN]` (needed for profiling metrics), default port 9400
2. In `prometheus.yml`, change the job target to `dcgm-exporter:9400`
3. `docker compose stop/rm` the old container, `docker compose up -d dcgm-exporter`, then `docker restart prometheus` to pick up the new scrape config
4. Switch Grafana to the official NVIDIA dashboard (ID 12239) and delete the old `Nvidia GPU Metrics` one (14574)

## DCP profiling module not loaded

```
Not collecting DCP metrics: ... module ... not currently loaded
```

The DCP (DCGM Cloud Profiling) module provides Tensor Core utilization, SM occupancy, memory bandwidth utilization, and other advanced profiling metrics. Investigation:

- Ruled out permissions: the container runs as root with `cap_sys_admin`, and `/dev/nvidia-caps` was mounted correctly
- Ruled out a driver restriction: `cat /proc/driver/nvidia/params | grep RmProfilingAdminOnly` showed 1, but since the container itself runs as root/admin, this wasn't the blocker
- The actual cause: the image in use at the time, `dcgm-exporter:3.3.5-3.4.1-ubuntu22.04` (DCGM 3.3.5, built ~Feb 2024), predates the host's **RTX PRO 6000 Blackwell Server Edition** GPUs — that DCGM version has no profiling counter definitions for this GPU generation, so it silently skips loading the DCP module
- Fix: upgrade to `nvcr.io/nvidia/k8s/dcgm-exporter:4.2.3-4.1.3-ubuntu22.04` (DCGM 4.1.3)

⚠️ **Watch out for the NGC `latest` tag trap**: this repository's `latest` tag turned out to be DCGM 2.3.4 (built Feb 2022) — older than 3.3.5. Many enterprise NGC images leave `latest` pointing at an early legacy build and only publish versioned tags going forward; `latest` is never updated again. **Always pin an explicit version** — don't assume `latest` means "newest."

The base OS in the tag (`ubuntu22.04`) is just the container's own userspace and doesn't need to match the host OS — the GPU driver is mounted into the container from the host via `nvidia-container-toolkit`/CDI, and the container doesn't ship its own driver. This host runs Ubuntu 24.04, and the container's `ubuntu22.04` base image works fine regardless.

## Tensor Core panel still empty after upgrading DCGM

Upgrading the image alone wasn't enough — dcgm-exporter still defaults to loading only `/etc/dcgm-exporter/default-counters.csv` (the basic metric set, no profiling fields). The image actually ships `/etc/dcgm-exporter/dcp-metrics-included.csv` too (includes `DCGM_FI_PROF_PIPE_TENSOR_ACTIVE`, `DCGM_FI_PROF_GR_ENGINE_ACTIVE`, etc.), which needs to be selected explicitly:

```yaml
environment:
  - DCGM_EXPORTER_COLLECTORS=/etc/dcgm-exporter/dcp-metrics-included.csv
```

After setting this and restarting the container, the `DCGM_FI_PROF_*` metrics show up in `/metrics`.

## `rule_files` mount added but the rule list is still empty

A variant of the [transient DNS error above](#prometheus-scrape-shows-a-transient-dns-error): after adding the `./prometheus/rules:/etc/prometheus/rules:ro` volume mount, a plain `docker restart prometheus` isn't enough — **adding or changing a volume mount requires `docker compose up -d <service>` to recreate the container**; `docker restart` just restarts the same container process and won't see the new mount. Check with `docker exec prometheus ls /etc/prometheus/rules/` to confirm the files are actually visible.

## Alertmanager container keeps restarting: permission denied reading its config

```
open /etc/alertmanager/alertmanager.yml: permission denied
```

`alertmanager.yml` contains a plaintext SMTP password, so it was locked down with `chmod 600` (owned by the deploying user) — but the Alertmanager image runs as a non-root user by default and can't read it.
**Don't fix this by loosening the file to 644/666** — on a shared server that exposes the SMTP password to every local user, i.e. a credential leak. The correct fix is to run the container as the host's matching UID, keeping the file at 600:

```yaml
alertmanager:
  user: "1000:1000" # replace with the host user's uid:gid (from `id -u` / `id -g`)
```

## How to verify the alert email actually got sent

At Alertmanager's default log level, **only failed sends are logged, not successful ones** — the container logs give no direct evidence either way. Verification steps:

```bash
# 1. First confirm network connectivity to the SMTP server
docker exec alertmanager sh -c "nc -zv smtp.mail.me.com 587"

# 2. Spin up a temporary instance with --log.level=debug and POST a test alert manually
docker run -d --name alertmanager-debug --network monitoring_default --user 1000:1000 \
  --tmpfs /alertmanager-data:uid=1000,gid=1000 \
  -v "$(pwd)/alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro" \
  prom/alertmanager:latest --config.file=/etc/alertmanager/alertmanager.yml \
  --storage.path=/alertmanager-data --log.level=debug

docker exec alertmanager-debug wget -qO- \
  --post-data='[{"labels":{"alertname":"Test","severity":"warning","instance":"test"},"annotations":{"summary":"debug test"}}]' \
  --header="Content-Type: application/json" http://localhost:9093/api/v2/alerts

# after group_wait (default 30s), check the logs — "Notify success" confirms the email really went out
docker logs alertmanager-debug 2>&1 | grep -i notify

# 3. Clean up the debug instance and restore the compose-managed one
docker rm -f alertmanager-debug
docker compose up -d alertmanager
```
