# LGTM Observability Stack — Docker-Based Monitoring Agent Deployment

A **safe deployment approach** for the monitoring agent: run Grafana Alloy inside Docker on each Laravel EC2 instance, while your application (Nginx, PHP-FPM, Laravel) continues to run natively via `systemctl`.

> **Key Principle**: If monitoring breaks → `docker compose down` — your application is **completely unaffected**. No downtime, no risk.

> [!IMPORTANT]
> This guide covers **only the Laravel EC2 side** (the monitoring agent). The LGTM server setup remains unchanged — follow the main [README.md](README.md) for the server side (Sections 1–3 and 6–8).

---

## What Changes vs. the Original Approach

| Component          | Original (README.md)                        | This Guide (Docker)                               |
| ------------------ | ------------------------------------------- | ------------------------------------------------- |
| **Alloy**          | Installed via `apt`, managed by `systemctl` | Runs in a Docker container                        |
| **System metrics** | Alloy reads `/proc` and `/sys` directly     | Alloy reads host `/proc`/`/sys` via volume mounts |
| **Nginx**          | Runs via `systemctl` (untouched)            | **Same — untouched**                              |
| **PHP-FPM**        | Runs via `systemctl` (untouched)            | **Same — untouched**                              |
| **Laravel app**    | Runs natively (untouched)                   | **Same — untouched**                              |
| **OpenTelemetry**  | Installed via Composer (untouched)          | **Same — untouched**                              |
| **Rollback**       | `sudo systemctl stop alloy`                 | `docker compose down`                             |

> **Why host networking?** The sidecar exporters (nginx-exporter, phpfpm-exporter) need to reach `127.0.0.1:8080/nginx_status` and `127.0.0.1:8080/fpm-status`. Alloy then scrapes those exporters on `127.0.0.1:9113` and `127.0.0.1:9253`. With `network_mode: host`, all containers share the host's network stack, so localhost endpoints work exactly like the systemd version.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Install Docker on Each Laravel EC2](#2-install-docker-on-each-laravel-ec2)
3. [⚠️ Enable Status Endpoints (Nginx & PHP-FPM)](#️-3-enable-status-endpoints-nginx--php-fpm) — _modifies host_
4. [Deploy the Monitoring Agent](#4-deploy-the-monitoring-agent)
5. [⚠️ Laravel Application Integration](#️-5-laravel-application-integration) — _modifies Laravel app_
6. [Operations & Management](#6-operations--management)
7. [Migration from Systemd Alloy](#7-migration-from-systemd-alloy)
8. [Verification & Testing](#8-verification--testing)
9. [Troubleshooting](#9-troubleshooting)

---

## 1. Prerequisites

Before starting, ensure you have completed these from the main [README.md](README.md):

- [ ] AWS prerequisites done (S3 bucket, IAM role, security groups) — [Section 1](README.md#1-aws-prerequisites)
- [ ] S3 lifecycle policy applied — [Section 2](README.md#2-s3-lifecycle-policy)
- [ ] LGTM server deployed and healthy (`docker compose up -d`) — [Section 3](README.md#3-lgtm-server-ec2-setup)
- [ ] Grafana accessible at `http://<LGTM-IP>:3000`

> The LGTM server setup is **identical** between this guide and the original. Only the Laravel EC2 agent deployment differs.

---

## 2. Install Docker on Each Laravel EC2

SSH into **each of the 6 Laravel EC2 instances** and run:

```bash
# Update the system
sudo apt update && sudo apt upgrade -y

# Install Docker (official script)
curl -fsSL https://get.docker.com | sudo sh

# Add your user to the docker group (avoids needing sudo for docker commands)
sudo usermod -aG docker $USER

# Install Docker Compose plugin
sudo apt install -y docker-compose-plugin

# Verify installation
docker --version
docker compose version

# IMPORTANT: Log out and back in for group membership to take effect
exit
```

After logging back in, verify Docker works without `sudo`:

```bash
docker run --rm hello-world
# Should print "Hello from Docker!"
```

---

## ⚠️ 3. Enable Status Endpoints (Nginx & PHP-FPM)

> [!CAUTION]
> **This section modifies the host directly (outside Docker).** It changes Nginx and PHP-FPM configuration files on the Laravel EC2. These are the **only changes made to the host** (besides installing Docker).

> This step is **identical** to [Section 4.2 of the main README](README.md#42-enable-nginx--php-fpm-status-endpoints). If you've already done this, skip ahead.

### 3.1 Nginx `stub_status`

```bash
sudo tee /etc/nginx/conf.d/stub_status.conf << 'EOF'
server {
    listen 8080;
    server_name localhost;
    location /nginx_status {
        stub_status on;
        allow 127.0.0.1;
        deny all;
    }
}
EOF
sudo nginx -t && sudo systemctl reload nginx

# Verify: should return Active connections, accepts, handled, requests
curl -s http://127.0.0.1:8080/nginx_status
```

### 3.2 PHP-FPM Status Page

```bash
# Enable the status path in PHP-FPM pool config
sudo sed -i 's#^;*pm.status_path = .*#pm.status_path = /fpm-status#' /etc/php/*/fpm/pool.d/www.conf
sudo systemctl restart php*-fpm

# Rewrite the dedicated status server so it exposes both endpoints on 127.0.0.1:8080
# IMPORTANT: Do NOT use "include fastcgi_params" here — duplicate SCRIPT_NAME values
# cause PHP-FPM to ignore the status path. Specify only the params needed.
# Adjust the PHP-FPM socket path if your host differs (check: ls /run/php/).
sudo tee /etc/nginx/conf.d/stub_status.conf << 'EOF'
server {
    listen 8080;
    server_name localhost;

    location = /nginx_status {
        stub_status on;
        allow 127.0.0.1;
        deny all;
    }

    location = /fpm-status {
        fastcgi_pass unix:/run/php/php-fpm.sock;    # Adjust to your PHP-FPM socket
        fastcgi_param SCRIPT_NAME     /fpm-status;
        fastcgi_param SCRIPT_FILENAME /fpm-status.php;
        fastcgi_param REQUEST_URI     /fpm-status;
        fastcgi_param QUERY_STRING    $query_string;
        fastcgi_param REQUEST_METHOD  $request_method;
        allow 127.0.0.1;
        deny all;
    }
}
EOF
sudo nginx -t && sudo systemctl reload nginx

# Verify: should return PHP-FPM pool status (pool: www, process manager: dynamic, ...)
curl -s http://127.0.0.1:8080/fpm-status
```

---

## 4. Deploy the Monitoring Agent

### 4.1 Copy the Configuration Files

```bash
# Clone or copy this repo to the server
cd /opt
sudo mkdir -p monitoring && sudo chown $USER:$USER monitoring
git clone <your-repo-url> monitoring
# Or: scp -r alloy-docker/ user@laravel-ec2:/opt/monitoring/

cd /opt/monitoring/alloy-docker
```

### 4.2 Edit the Alloy Configuration

```bash
# Edit the Alloy config file
nano config.alloy
```

Replace these placeholders:

| Placeholder                  | Replace With                  | Example                                 |
| ---------------------------- | ----------------------------- | --------------------------------------- |
| `LGTM_SERVER_PRIVATE_IP`     | Your LGTM server's private IP | `10.0.1.50`                             |
| `instance = "laravel-app-1"` | Unique name for this EC2      | `laravel-app-1` through `laravel-app-6` |

> **There are 3 places** where you need to replace `LGTM_SERVER_PRIVATE_IP`:
>
> 1. `loki.write` → `url` (line ~87)
> 2. `prometheus.remote_write` → `url` (line ~158)
> 3. `otelcol.exporter.otlphttp` → `endpoint` (line ~213)

> **There are 2 places** where you need to set the instance name:
>
> 1. `loki.process` → `stage.static_labels` → `instance` (line ~48)
> 2. `prometheus.relabel` → `rule` → `replacement` (line ~147)

### 4.3 Verify Log Paths

Make sure the Laravel log directory path matches your actual setup:

```bash
# Check where your Laravel logs are
ls -la /var/www/html/storage/logs/

# If your Laravel app is in a different directory (e.g., /var/www/your-app),
# update both docker-compose.yml volumes AND config.alloy paths accordingly
```

### 4.4 Start the Monitoring Agent

```bash
cd /opt/monitoring/alloy-docker

# Start in detached mode
docker compose up -d

# Watch the logs to ensure Alloy starts cleanly
docker compose logs -f --tail=50
# Press Ctrl+C to stop following logs

# Verify all containers are healthy
docker compose ps
# Expected:
# NAME               STATUS         PORTS
# alloy              Up (healthy)
# nginx-exporter     Up
# phpfpm-exporter    Up
```

### 4.5 Verify Alloy Is Running

```bash
# Alloy's debug UI is accessible at:
# http://localhost:12345 (from the EC2 itself)
# If the page loads (returns HTML), Alloy is running.
curl -s http://localhost:12345/ | head -1
# Expected: HTML page (<!doctype html>...)
```

---

## ⚠️ 5. Laravel Application Integration

> [!CAUTION]
> **This section modifies the Laravel application directly (outside Docker).** It installs Composer packages, edits `.env`, and registers middleware. Your application code is being changed here.

This section is **identical** to [Section 5 of the main README](README.md#5-laravel-application-integration). Because Alloy uses host networking, your Laravel app sends traces to `localhost:4318` — exactly the same as the systemd approach.

### 5.1 Install OpenTelemetry (Tracing)

On each Laravel EC2, run as the app user (e.g., `theone`) in the Laravel project directory:

```bash
cd /home/theone/kol

# Core OpenTelemetry SDK + OTLP exporter
composer require open-telemetry/sdk open-telemetry/exporter-otlp

# Laravel auto-instrumentation (recommended)
composer require keepsuit/laravel-opentelemetry

# Publish the config
php artisan vendor:publish --provider="Keepsuit\LaravelOpenTelemetry\LaravelOpenTelemetryServiceProvider"
```

> **Note**: Adjust `/home/theone/kol` to your actual Laravel project path. Run as the user that owns the project directory to preserve correct file permissions.

### 5.2 Configure OpenTelemetry Environment

Append the OpenTelemetry settings to your Laravel `.env` file (reference: [`laravel/.env.otel.example`](laravel/.env.otel.example)):

```bash
cat >> /home/theone/kol/.env << 'EOF'

# OpenTelemetry
OTEL_SERVICE_NAME=duadualive-staging
OTEL_TRACES_EXPORTER=otlp
OTEL_METRICS_EXPORTER=otlp
OTEL_LOGS_EXPORTER=none
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
OTEL_PHP_AUTOLOAD_ENABLED=true
OTEL_RESOURCE_ATTRIBUTES=deployment.environment=production,service.namespace=laravel
OTEL_TRACES_SAMPLER=parentbased_traceidratio
OTEL_TRACES_SAMPLER_ARG=1.0
EOF
```

> **Notice**: `OTEL_EXPORTER_OTLP_ENDPOINT` still points to `http://localhost:4318` — this works because Alloy uses host networking. No change from the systemd approach.
>
> **Adjust** `OTEL_SERVICE_NAME` to a unique name per instance (e.g., `duadualive-staging`, `laravel-app-2`, etc.).

### 5.3 Install TraceId Middleware (Log ↔ Trace Correlation)

```bash
# Copy the reference middleware (adjust the destination to your Laravel project path)
cp laravel/TraceIdMiddleware.php /home/theone/kol/app/Http/Middleware/TraceIdMiddleware.php
```

Register it in `bootstrap/app.php` (Laravel 11+):

```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->append(\App\Http\Middleware\TraceIdMiddleware::class);
})
```

Or for Laravel 10, add to `$middleware` in `app/Http/Kernel.php`:

```php
protected $middleware = [
    // ... existing middleware
    \App\Http\Middleware\TraceIdMiddleware::class,
];
```

> See [`laravel/README.md`](laravel/README.md) for the complete Laravel integration guide.

---

## 6. Operations & Management

### 6.1 Common Docker Commands

```bash
cd /opt/monitoring/alloy-docker

# View container status
docker compose ps

# View live logs
docker compose logs -f alloy

# Restart Alloy (e.g., after config changes)
docker compose restart alloy

# Stop monitoring (app keeps running!)
docker compose down

# Start monitoring again
docker compose up -d

# Update Alloy to a newer version
# 1. Edit docker-compose.yml → change the image tag
# 2. Then:
docker compose pull
docker compose up -d
```

### 6.2 Updating the Alloy Configuration

```bash
# Edit the config
nano /opt/monitoring/alloy-docker/config.alloy

# Restart Alloy to apply changes
docker compose restart alloy

# Verify it's healthy
docker compose ps
docker compose logs --tail=20 alloy
```

### 6.3 Auto-Start on Boot

Docker's `restart: unless-stopped` policy ensures the monitoring container starts automatically when the EC2 reboots. Verify Docker itself starts on boot:

```bash
# Ensure Docker daemon starts on boot
sudo systemctl enable docker

# Verify
sudo systemctl is-enabled docker
# Expected: enabled
```

### 6.4 Monitoring Resource Usage

```bash
# Check how much CPU/memory Alloy is using
docker stats alloy --no-stream

# Expected: Alloy typically uses ~50-150 MB RAM and <5% CPU
```

---

## 7. Migration from Systemd Alloy

If you previously installed Alloy via `apt` (the systemd approach from the main README), follow these steps to migrate to Docker:

### 7.1 Stop and Disable Systemd Alloy

```bash
# Stop the existing Alloy service
sudo systemctl stop alloy

# Disable it so it doesn't start on boot
sudo systemctl disable alloy

# Verify it's stopped
sudo systemctl status alloy
# Expected: inactive (dead)
```

### 7.2 Deploy the Docker Version

Follow [Section 4](#4-deploy-the-monitoring-agent) above.

### 7.3 Verify Data Flow Continues

```bash
# Check Alloy container is healthy
docker compose ps

# Check readiness
curl -s http://localhost:12345/ready

# Verify in Grafana that data from this instance is still flowing:
# - Explore → Loki → {instance="laravel-app-1"} → should see new logs
# - Explore → Mimir → up{instance="laravel-app-1"} → should show 1
```

### 7.4 (Optional) Clean Up Systemd Alloy

Once you've confirmed the Docker version is working:

```bash
# Remove the apt-installed Alloy
sudo apt remove -y alloy

# Remove leftover config
sudo rm -rf /etc/alloy/

# Clean up the apt repository (optional)
sudo rm /etc/apt/sources.list.d/grafana.list
```

> **Rollback plan**: If the Docker version doesn't work, simply `docker compose down`, re-enable the systemd version with `sudo systemctl enable --now alloy`, and you're back to the original setup.

---

## 8. Verification & Testing

### 8.1 Verify the Alloy Container

```bash
# Container should be running and healthy
docker compose ps
# Expected:
# NAME               STATUS         PORTS
# alloy              Up (healthy)
# nginx-exporter     Up
# phpfpm-exporter    Up

# Check for errors in the logs
docker compose logs --tail=50 alloy

# Alloy readiness check (returns HTML if running)
curl -s http://localhost:12345/ | head -1
# Expected: <!doctype html>...
```

### 8.2 Verify System Metrics Are Being Collected

```bash
# Check that Alloy can read host /proc and /sys
docker exec alloy ls /host/proc/stat
# Expected: /host/proc/stat

docker exec alloy ls /host/sys/class/net
# Expected: list of network interfaces
```

### 8.3 Verify Status Endpoints and Exporters Are Reachable

```bash
# Verify the raw status pages are reachable
curl -s http://127.0.0.1:8080/nginx_status
# Expected: Active connections: ...

curl -s http://127.0.0.1:8080/fpm-status
# Expected: pool, process manager, ...

# Verify the exporter containers are exposing Prometheus metrics
curl -s http://127.0.0.1:9113/metrics | head -5
# Expected: lines starting with nginx_...

curl -s http://127.0.0.1:9253/metrics | head -5
# Expected: lines starting with phpfpm_...
```

### 8.4 Verify Data Flow in Grafana

From the LGTM server:

```bash
# Query recent logs from this instance (adjust instance name to match your config)
curl -s 'http://localhost:3100/loki/api/v1/query?query={instance="duadualive-staging"}&limit=5' | jq .

# Query metrics
curl -s 'http://localhost:9009/prometheus/api/v1/query?query=up%7Binstance%3D%22duadualive-staging%22%7D' | jq .

# Search for recent traces
curl -s 'http://localhost:3200/api/search?limit=5' | jq .
```

Or verify in **Grafana UI** at `http://<LGTM-IP>:3000`:

1. **Explore → Loki** → `{instance="duadualive-staging"}` → should see log entries
2. **Explore → Mimir** → `up{instance="duadualive-staging"}` → should show `1`
3. **Explore → Tempo** → Search → should see traces (after OpenTelemetry is configured)

### 8.5 End-to-End Smoke Test

Run this from the Laravel EC2:

```bash
# Generate a test log entry
echo "[$(date '+%Y-%m-%d %H:%M:%S')] production.ERROR: Docker smoke test from $(hostname)" \
  >> /home/theone/kol/storage/logs/laravel.log

# Generate a test trace (via OTLP HTTP)
curl -X POST http://localhost:4318/v1/traces \
  -H "Content-Type: application/json" \
  -d '{
    "resourceSpans": [{
      "resource": {
        "attributes": [{"key": "service.name", "value": {"stringValue": "smoke-test-docker"}}]
      },
      "scopeSpans": [{
        "scope": {"name": "test"},
        "spans": [{
          "traceId": "'"$(openssl rand -hex 16)"'",
          "spanId": "'"$(openssl rand -hex 8)"'",
          "name": "docker-smoke-test-span",
          "kind": 1,
          "startTimeUnixNano": "'"$(date +%s)000000000"'",
          "endTimeUnixNano": "'"$(( $(date +%s) + 1 ))000000000"'",
          "status": {"code": 1}
        }]
      }]
    }]
  }'

# Wait 30 seconds, then verify in Grafana:
# 1. Explore → Loki → {job="laravel", level="ERROR"} → should see "Docker smoke test"
# 2. Explore → Tempo → Search → should see "docker-smoke-test-span"
# 3. Explore → Mimir → query "up" → should see this instance
```

---

## File Structure

```
log-monitoring/
├── README.md                          ← Main guide (systemd approach)
├── README-docker.md                   ← This file (Docker approach)
├── aws/
│   ├── iam-policy-lgtm-s3.json       ← IAM policy for S3 access
│   └── s3-lifecycle-policy.json       ← S3 tiering & retention
├── lgtm-server/
│   ├── docker-compose.yml             ← Full LGTM stack deployment
│   ├── loki-config.yaml               ← Loki configuration
│   ├── mimir-config.yaml              ← Mimir configuration
│   ├── tempo-config.yaml              ← Tempo configuration
│   ├── alertmanager-fallback-config.yaml
│   └── grafana/
│       └── provisioning/
│           └── datasources/
│               └── datasources.yaml   ← Auto-provisioned datasources
├── alloy/
│   └── config.alloy                   ← Alloy config (systemd version)
├── alloy-docker/                      ← NEW: Docker-based Alloy deployment
│   ├── docker-compose.yml             ← Alloy container definition
│   └── config.alloy                   ← Alloy config (Docker version)
└── laravel/
    ├── README.md                      ← Laravel integration guide
    ├── TraceIdMiddleware.php           ← Reference: log↔trace correlation middleware
    └── .env.otel.example              ← Reference: OpenTelemetry .env variables
```

---

## Quick Reference — Post-Deployment Checklist

### LGTM Server (same as main README)

- [ ] S3 bucket created with encryption and public access blocked
- [ ] Lifecycle policy applied (30d Standard → Intelligent-Tiering → 100d delete)
- [ ] IAM Instance Profile attached to LGTM EC2
- [ ] Security groups configured (Loki:3100, Mimir:9009, Tempo:4317/4318, Grafana:3000)
- [ ] Docker and Docker Compose installed on LGTM EC2
- [ ] `docker compose up -d` — all services healthy
- [ ] Grafana accessible at `http://<LGTM-IP>:3000`

### Laravel EC2s (Docker approach)

- [ ] Docker and Docker Compose installed on all 6 Laravel EC2 instances
- [ ] Docker daemon enabled on boot (`systemctl enable docker`)
- [ ] Nginx `stub_status` enabled on each Laravel EC2
- [ ] PHP-FPM `pm.status_path` enabled on each Laravel EC2
- [ ] Alloy config updated with correct LGTM server IP and unique instance names
- [ ] `docker compose up -d` — Alloy container is healthy
- [ ] OpenTelemetry packages installed in Laravel (`keepsuit/laravel-opentelemetry`)
- [ ] `.env` updated with OTEL variables on each instance
- [ ] TraceId middleware registered in Laravel
- [ ] Logs visible in Grafana → Explore → Loki
- [ ] Metrics visible in Grafana → Explore → Mimir
- [ ] Traces visible in Grafana → Explore → Tempo (with request waterfalls)
- [ ] Log ↔ Trace correlation working (click traceId in log → opens trace)
- [ ] If migrating: old systemd Alloy stopped and disabled
- [ ] Grafana admin password changed from default
