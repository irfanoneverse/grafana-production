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

> This step is **identical** to [Section 4.2 of the main README](README.md#42-enable-nginx--php-fpm-status-endpoints). If you've already configured Nginx stub_status and PHP-FPM status endpoints, skip ahead to [Section 4](#4-deploy-the-monitoring-agent).

### 3.1 Enable PHP-FPM Status Path

```bash
# Enable the status path in PHP-FPM pool config
sudo sed -i 's#^;*pm.status_path = .*#pm.status_path = /fpm-status#' /etc/php/*/fpm/pool.d/www.conf
sudo systemctl restart php*-fpm
```

### 3.2 Create the Nginx Status Server

This creates a dedicated Nginx server on port 8080 that exposes both Nginx and PHP-FPM status endpoints.

> **IMPORTANT**: Do **not** use `include fastcgi_params` in the `/fpm-status` block — it overrides the `SCRIPT_NAME` parameter and causes PHP-FPM to return "File not found" or "Access denied". Only specify the exact params needed.

```bash
# Check your PHP-FPM socket path first (adjust the fastcgi_pass line below if different)
ls /run/php/

# Create the combined status server on port 8080
sudo tee /etc/nginx/conf.d/stub_status.conf > /dev/null << 'EOF'
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
```

### 3.3 Verify Both Endpoints

```bash
# Nginx status — should show: Active connections, accepts, handled, requests
curl -s http://127.0.0.1:8080/nginx_status

# PHP-FPM status — should show: pool, process manager, start time, ...
curl -s http://127.0.0.1:8080/fpm-status
```

> **Common issues**:
> - `File not found.` → `SCRIPT_NAME` mismatch with `pm.status_path`, or you used `include fastcgi_params`
> - `Access denied.` → `security.limit_extensions` blocking. Set `SCRIPT_FILENAME /fpm-status.php;` (must end in `.php`)
> - `502 Bad Gateway` → PHP-FPM socket path is wrong. Check `ls /run/php/` for the actual socket file (e.g., `php8.3-fpm.sock`)
> - `location directive not allowed here` → The `location` block must be inside a `server {}` block

### 3.4 Special Handling: Multiple PHP-FPM Versions

If you have multiple PHP versions (e.g., 8.3 and 8.4), follow these steps to ensure the correct status page is scraped:

**Step 1: Enable status path for all relevant versions**
```bash
# Enable in 8.3
sudo sed -i 's/^;\(pm\.status_path\)/\1/' /etc/php/8.3/fpm/pool.d/www.conf
sudo systemctl restart php8.3-fpm

# Enable in 8.4
sudo sed -i 's/^;\(pm\.status_path\)/\1/' /etc/php/8.4/fpm/pool.d/www.conf
sudo systemctl restart php8.4-fpm
```

**Step 2: Update Nginx to point to the active socket**
Check which socket your app uses (`ls /run/php/`) and update `/etc/nginx/conf.d/stub_status.conf`:
```bash
# Example: Point to PHP 8.3
sudo sed -i 's#unix:/run/php/php-fpm.sock#unix:/run/php/php8.3-fpm.sock#' /etc/nginx/conf.d/stub_status.conf
sudo nginx -t && sudo systemctl reload nginx

# Example: Switch to PHP 8.4
sudo sed -i 's#unix:/run/php/php8.3-fpm.sock#unix:/run/php/php8.4-fpm.sock#' /etc/nginx/conf.d/stub_status.conf
sudo systemctl reload nginx
```

**Step 3: Verify**
```bash
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

| Placeholder                  | Replace With                      | Example                                 |
| ---------------------------- | --------------------------------- | --------------------------------------- |
| `LGTM_SERVER_PRIVATE_IP`     | Your LGTM server's private IP     | `172.31.24.243`                          |
| `instance = "laravel-app-1"` | Unique name for this EC2          | `core-payment`, `duadualive-staging` |
| `location = "Asia/Kuala_Lumpur"` | Your server's timezone (IANA) | `Asia/Singapore`, `UTC`, `America/New_York` |

> **There are 3 places** where you need to replace `LGTM_SERVER_PRIVATE_IP`:
>
> 1. `loki.write` → `url` (line ~91)
> 2. `prometheus.remote_write` → `url` (line ~159)
> 3. `otelcol.exporter.otlphttp` → `endpoint` (line ~214)

> **There are 2 places** where you need to set the instance name:
>
> 1. `loki.process` → `stage.static_labels` → `instance` (line ~52)
> 2. `prometheus.relabel` → `rule` → `replacement` (line ~151)

> **Timezone**: The `stage.timestamp` block (line ~67) has `location = "Asia/Kuala_Lumpur"`. If your servers are in a different timezone, change this to the correct [IANA timezone](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones). **Getting this wrong causes Loki to reject all Laravel logs** with "timestamp too new" errors.

### 4.3 Verify Log Paths

Make sure the Laravel log directory path matches your actual setup. The default config uses `/home/forge/payment-api-ookwuvmk.on-forge.com/current/storage/logs/` — update if your app is in a different location:

```bash
# Find your Laravel project root (look for artisan, storage/, public/)
# Common locations: /home/<user>/<app>, /var/www/html, /var/www/<app>
ls -la /home/forge/payment-api-ookwuvmk.on-forge.com/current/storage/logs/

# If your path is DIFFERENT, update these 2 files:
# 1. config.alloy → local.file_match "laravel_logs" → __path__
# 2. docker-compose.yml → alloy volumes → Laravel log mount
#
# Example: if your app is at /var/www/myapp, change:
#   config.alloy:        "/var/www/myapp/storage/logs/*.log"
#    docker-compose.yml:  /home/forge/payment-api-ookwuvmk.on-forge.com/current/storage/logs:/home/forge/payment-api-ookwuvmk.on-forge.com/current/storage/logs:ro
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

On each Laravel EC2, run as the app user in the Laravel project directory:

```bash
cd /home/forge/payment-api-ookwuvmk.on-forge.com/current

# Core OpenTelemetry SDK + OTLP exporter
composer require open-telemetry/sdk open-telemetry/exporter-otlp

# Laravel auto-instrumentation (recommended)
composer require keepsuit/laravel-opentelemetry

# Publish the config
php artisan vendor:publish --provider="Keepsuit\LaravelOpenTelemetry\LaravelOpenTelemetryServiceProvider"
```

> **Note**: Adjust `/home/forge/payment-api-ookwuvmk.on-forge.com/current` to your actual Laravel project path. Run as the user that owns the project directory to preserve correct file permissions.

### 5.2 Configure OpenTelemetry Environment

Append the OpenTelemetry settings to your Laravel `.env` file (reference: [`laravel/.env.otel.example`](laravel/.env.otel.example)):

```bash
cat >> /home/forge/payment-api-ookwuvmk.on-forge.com/current/.env << 'EOF'

# OpenTelemetry
OTEL_SERVICE_NAME=core-payment
OTEL_TRACES_EXPORTER=otlp
OTEL_METRICS_EXPORTER=none
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
> **Adjust** `OTEL_SERVICE_NAME` to a unique name per instance (e.g., `core-payment`, `duadualive-staging`, etc.).

### 5.3 Install TraceId Middleware (Log ↔ Trace Correlation)

```bash
# Copy the reference middleware (adjust the destination to your Laravel project path)
cp laravel/TraceIdMiddleware.php /home/forge/payment-api-ookwuvmk.on-forge.com/current/app/Http/Middleware/TraceIdMiddleware.php
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

### 5.4 Automated Deployment (Middleware Injection)

To quickly deploy the middleware across your nodes, you can use these commands in your project root (`/home/forge/payment-api-ookwuvmk.on-forge.com/current`):

```bash
# 1. Copy the middleware
mkdir -p app/Http/Middleware
cp /opt/monitoring/laravel/TraceIdMiddleware.php app/Http/Middleware/TraceIdMiddleware.php

# 2. Inject into the application structure
if [ -f "bootstrap/app.php" ]; then
    echo "Detected Laravel 11 structure (bootstrap/app.php)"
    sed -i '/->withMiddleware(function (Middleware $middleware) {/a \        $middleware->append(\\App\\Http\\Middleware\\TraceIdMiddleware::class);' bootstrap/app.php
elif [ -f "app/Http/Kernel.php" ]; then
    echo "Detected Laravel 10 structure (app/Http/Kernel.php)"
    sed -i '/protected $middleware = \[/a \        \\App\\Http\\Middleware\\TraceIdMiddleware::class,' app/Http/Kernel.php
else
    echo "Middleware file not found. Manual update required."
fi

# 3. Clear configuration cache
php artisan config:clear
```

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

# Check Alloy is running (returns HTML debug UI)
curl -s http://localhost:12345/ | head -1

# Verify in Grafana that data from this instance is still flowing:
# - Explore → Loki → {instance="YOUR_INSTANCE"} → should see new logs
# - Explore → Mimir → up{instance="YOUR_INSTANCE"} → should show 1
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
curl -s 'http://localhost:3100/loki/api/v1/query?query={instance="core-payment"}&limit=5' | jq .

# Query metrics
curl -s 'http://localhost:9009/prometheus/api/v1/query?query=up%7Binstance%3D%22core-payment%22%7D' | jq .

# Search for recent traces
curl -s 'http://localhost:3200/api/search?limit=5' | jq .
```

Or verify in **Grafana UI** at `http://<LGTM-IP>:3000`:

1. **Explore → Loki** → `{instance="core-payment"}` → should see log entries
2. **Explore → Mimir** → `up{instance="core-payment"}` → should show `1`
3. **Explore → Tempo** → Search → should see traces (after OpenTelemetry is configured)

### 8.5 End-to-End Smoke Test

Run this from the Laravel EC2:

```bash
# Generate a test log entry (use sudo -u to match the Laravel app's file owner)
sudo -u someuser bash -c 'echo "['"$(date '+%Y-%m-%d %H:%M:%S')"'] production.ERROR: Docker smoke test from '"$(hostname)"'" >> /home/forge/payment-api-ookwuvmk.on-forge.com/current/storage/logs/laravel.log'

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

## 9. Troubleshooting

### 9.1 Loki Rejects Laravel Logs — "timestamp too new"

**Symptom**: Alloy logs show `status=400` errors:

```
error="server returned HTTP status 400 Bad Request (400): entry for stream '...' has timestamp too new: ..."
```

**Cause**: Laravel logs use server local time (e.g., `[2026-03-06 11:51:25]`) without timezone info. If `stage.timestamp` in `config.alloy` doesn't specify a `location`, Alloy parses the timestamp as UTC. For a UTC+8 server, this makes the timestamp appear 8 hours in the future — and Loki rejects it.

**Fix**: Ensure `config.alloy` has the correct timezone in `stage.timestamp`:

```
stage.timestamp {
    source   = "timestamp"
    format   = "2006-01-02 15:04:05"
    location = "Asia/Kuala_Lumpur"   // Change to your server's IANA timezone
}
```

### 9.2 Permission Denied Writing Test Log Entries

**Symptom**: `bash: /home/forge/payment-api-ookwuvmk.on-forge.com/current/storage/logs/laravel.log: Permission denied`

**Cause**: Log files are owned by the Laravel app user (e.g., `www-data` or your deploy user), not the `ubuntu` SSH user.

**Fix**: Use `sudo -u` to write as the app user:

```bash
sudo -u someuser bash -c 'echo "['"$(date '+%Y-%m-%d %H:%M:%S')"'] production.ERROR: Test" >> /home/forge/payment-api-ookwuvmk.on-forge.com/current/storage/logs/laravel.log'
```

### 9.3 Alloy Shows "component does not exist" for Nginx/PHP-FPM

**Symptom**: Alloy logs show:

```
cannot find the definition of component name "prometheus.exporter.nginx"
cannot find the definition of component name "prometheus.exporter.php_fpm"
```

**Cause**: Alloy does **not** have built-in Nginx or PHP-FPM exporters. These are separate processes.

**Fix**: Use the sidecar exporter containers in `docker-compose.yml` (already included). Alloy scrapes them via `prometheus.scrape` on `:9113` (Nginx) and `:9253` (PHP-FPM).

### 9.4 Pulling Past History Logs into Loki

By default, `tail_from_end = true` means Alloy only reads **new** log lines. If you need to ingest the historical logs (e.g., for `core-payment` initial setup), follow this sequence:

**Step 1: Set Alloy to read from start**
In `alloy-docker/config.alloy`, set:
```alloy
tail_from_end = false
```

**Step 2: Reset the "Bookmark" (Position File)**
Alloy remembers where it stopped via a position file stored in a Docker volume. To reset it:
```bash
cd /opt/monitoring/alloy-docker

# Stop the containers
docker compose down

# Delete the position data volume (clears the bookmark)
docker run --rm -v alloy-docker_alloy-data:/data alpine sh -c "rm -rf /data/*"
```

**Step 3: Start Ingestion**
```bash
docker compose up -d

# Monitor the progress (should see old logs being processed)
docker compose logs -f alloy
```
Once the logs from "today" appear in Grafana, the backfill is complete.

**Step 4: Restore Normal Mode**
To prevent accidental re-ingestion if the volume is ever lost, change `alloy-docker/config.alloy` back to:
```alloy
tail_from_end = true
```
Then restart without clearing data:
```bash
docker compose restart alloy
```

> ⚠️ **Note**: Logs older than the `reject_old_samples_max_age` configured on the Loki server (default is often 7 days/168h) will be silently rejected. Ensure your Loki server config allows the age of logs you are trying to backfill.

### 9.5 Useful Loki Queries

Use these in **Grafana → Explore → Loki**:

| Query | What it shows |
|---|---|
| `{job="laravel", instance="YOUR_INSTANCE"}` | All Laravel application logs |
| `{job="laravel", instance="YOUR_INSTANCE", level="ERROR"}` | Laravel errors only |
| `{job="laravel", instance="YOUR_INSTANCE", level="WARNING"}` | Laravel warnings only |
| `{job="nginx", instance="YOUR_INSTANCE"}` | Nginx access + error logs |
| `{job="php-fpm", instance="YOUR_INSTANCE"}` | PHP-FPM process logs |
| `{job="syslog", instance="YOUR_INSTANCE"}` | System logs (OS-level) |
| `{instance="YOUR_INSTANCE"} \|= "ERROR"` | Text search for "ERROR" across all logs |
| `{job="laravel"} \| logfmt \| line_format "{{.level}}: {{.message}}"` | Formatted output |

### 9.6 No Data in Grafana After Deployment

**Check connectivity** from the Laravel EC2 to the LGTM server:

```bash
curl -s -o /dev/null -w "%{http_code}" http://LGTM_SERVER_IP:3100/ready   # Loki
curl -s -o /dev/null -w "%{http_code}" http://LGTM_SERVER_IP:9009/ready   # Mimir
curl -s -o /dev/null -w "%{http_code}" http://LGTM_SERVER_IP:4318/v1/traces  # Tempo
```

- `200` / `405` = port is reachable ✅
- `000` = connection refused/timeout → **check AWS security groups** (LGTM server must allow inbound from Laravel SG on ports 3100, 9009, 4317, 4318)

**Check Alloy push errors**:

```bash
docker compose logs --tail=50 alloy 2>&1 | grep -i -E "error|fail|refused|timeout"
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
├── alloy-docker/                      ← Docker-based Alloy deployment (this guide)
│   ├── docker-compose.yml             ← Alloy + sidecar exporters container definitions
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
- [ ] Nginx `stub_status` enabled on port 8080 (`/nginx_status`)
- [ ] PHP-FPM `pm.status_path = /fpm-status` enabled (no `include fastcgi_params`!)
- [ ] Both status pages verified: `curl http://127.0.0.1:8080/nginx_status` and `curl http://127.0.0.1:8080/fpm-status`
- [ ] Alloy config updated: LGTM server IP, unique instance names, correct timezone, correct Laravel log path
- [ ] `docker compose up -d` — all 3 containers healthy (alloy, nginx-exporter, phpfpm-exporter)
- [ ] OpenTelemetry packages installed in Laravel (`keepsuit/laravel-opentelemetry`)
- [ ] `.env` updated with OTEL variables on each instance
- [ ] TraceId middleware registered in Laravel
- [ ] Logs visible in Grafana → Explore → Loki
- [ ] Metrics visible in Grafana → Explore → Mimir
- [ ] Traces visible in Grafana → Explore → Tempo (with request waterfalls)
- [ ] Log ↔ Trace correlation working (click traceId in log → opens trace)
- [ ] If migrating: old systemd Alloy stopped and disabled
- [ ] Grafana admin password changed from default
