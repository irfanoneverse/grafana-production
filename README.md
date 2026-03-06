# LGTM Observability Stack — Production Implementation Guide

A complete, production-ready centralized observability platform using **Loki** (logs), **Grafana** (visualization), **Tempo** (traces), and **Mimir** (metrics), with **Grafana Alloy** as the unified collector agent on each application node.

> **Gold Standard Approach**: This setup uses Alloy's **built-in exporters** instead of standalone binaries, and **OpenTelemetry auto-instrumentation** for deep request-level tracing. Zero extra systemd services beyond Alloy itself.

**Key difference from traditional setups**: Alloy is the **only process** you install. It replaces Node Exporter, Nginx Exporter, and PHP-FPM Exporter with built-in equivalents, while also collecting logs and forwarding traces.

---

## Table of Contents

1. [AWS Prerequisites](#1-aws-prerequisites)
2. [S3 Lifecycle Policy](#2-s3-lifecycle-policy)
3. [LGTM Server EC2 Setup](#3-lgtm-server-ec2-setup)
4. [Grafana Alloy Setup (Laravel EC2s)](#4-grafana-alloy-setup-on-each-laravel-ec2)
5. [Laravel Application Integration](#5-laravel-application-integration)
6. [Grafana Configuration](#6-grafana-configuration)
7. [Security & Networking](#7-security--networking)
8. [Verification & Testing](#8-verification--testing)

---

## 1. AWS Prerequisites

### 1.1 S3 Bucket

Create a single S3 bucket for all observability data. Each component uses a different prefix internally.

```bash
# Create the bucket (change region as needed)
aws s3api create-bucket \
  --bucket YOUR_COMPANY-observability-data \
  --region ap-southeast-1 \
  --create-bucket-configuration LocationConstraint=ap-southeast-1

# Enable versioning (recommended for data safety)
aws s3api put-bucket-versioning \
  --bucket YOUR_COMPANY-observability-data \
  --versioning-configuration Status=Enabled

# Block ALL public access
aws s3api put-public-access-block \
  --bucket YOUR_COMPANY-observability-data \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

# Enable server-side encryption (SSE-S3)
aws s3api put-bucket-encryption \
  --bucket YOUR_COMPANY-observability-data \
  --server-side-encryption-configuration '{
    "Rules": [{"ApplyServerSideEncryptionByDefault": {"SSEAlgorithm": "AES256"}}]
  }'
```

### 1.2 IAM Role for the LGTM Server EC2

The LGTM EC2 needs read/write access to S3. We use an **IAM Instance Profile** (no access keys stored on disk).

#### Step 1: Create the IAM Policy

The policy file is at [`aws/iam-policy-lgtm-s3.json`](aws/iam-policy-lgtm-s3.json).

```bash
aws iam create-policy \
  --policy-name LGTMStackS3Access \
  --policy-document file://aws/iam-policy-lgtm-s3.json
```

#### Step 2: Create the IAM Role and attach the policy

```bash
# Create a trust policy for EC2
cat > /tmp/ec2-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "ec2.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create the role
aws iam create-role \
  --role-name LGTMServerRole \
  --assume-role-policy-document file:///tmp/ec2-trust-policy.json

# Attach the S3 policy (use your account ID)
aws iam attach-role-policy \
  --role-name LGTMServerRole \
  --policy-arn arn:aws:iam::YOUR_ACCOUNT_ID:policy/LGTMStackS3Access

# Create the instance profile and add the role
aws iam create-instance-profile --instance-profile-name LGTMServerProfile
aws iam add-role-to-instance-profile \
  --instance-profile-name LGTMServerProfile \
  --role-name LGTMServerRole
```

#### Step 3: Attach the Instance Profile to the LGTM EC2

```bash
aws ec2 associate-iam-instance-profile \
  --instance-id i-0xxxxxxxxxxLGTM \
  --iam-instance-profile Name=LGTMServerProfile
```

> **IAM for Laravel EC2s**: The Laravel instances do **not** need any S3 IAM role. They only communicate with the LGTM stack over HTTP on the private network. No cloud credentials are needed on these nodes.

### 1.3 Security Groups

You need two security groups:

#### Security Group: `sg-lgtm-server`

Applied to the **LGTM EC2 instance**. Allows inbound from the Laravel instances.

| Port | Protocol | Source             | Service         | Purpose                    |
| ---- | -------- | ------------------ | --------------- | -------------------------- |
| 3100 | TCP      | `sg-laravel-app`   | Loki HTTP API   | Alloy pushes logs          |
| 9009 | TCP      | `sg-laravel-app`   | Mimir HTTP API  | Alloy pushes metrics       |
| 4317 | TCP      | `sg-laravel-app`   | Tempo OTLP gRPC | Alloy pushes traces (gRPC) |
| 4318 | TCP      | `sg-laravel-app`   | Tempo OTLP HTTP | Alloy pushes traces (HTTP) |
| 3000 | TCP      | Your IP / VPN CIDR | Grafana UI      | Web dashboard access       |
| 22   | TCP      | Your IP / VPN CIDR | SSH             | Administration             |

```bash
# Create the security group
LGTM_SG=$(aws ec2 create-security-group \
  --group-name sg-lgtm-server \
  --description "LGTM Observability Stack" \
  --vpc-id vpc-xxxxxxxx \
  --query 'GroupId' --output text)

# Get the Laravel security group ID (assumes it already exists)
LARAVEL_SG="sg-xxxxxxxxLARAVEL"

# Loki (logs)
aws ec2 authorize-security-group-ingress --group-id $LGTM_SG \
  --protocol tcp --port 3100 --source-group $LARAVEL_SG

# Mimir (metrics)
aws ec2 authorize-security-group-ingress --group-id $LGTM_SG \
  --protocol tcp --port 9009 --source-group $LARAVEL_SG

# Tempo OTLP gRPC (traces)
aws ec2 authorize-security-group-ingress --group-id $LGTM_SG \
  --protocol tcp --port 4317 --source-group $LARAVEL_SG

# Tempo OTLP HTTP (traces)
aws ec2 authorize-security-group-ingress --group-id $LGTM_SG \
  --protocol tcp --port 4318 --source-group $LARAVEL_SG

# Grafana UI (from your VPN/IP only)
aws ec2 authorize-security-group-ingress --group-id $LGTM_SG \
  --protocol tcp --port 3000 --cidr YOUR_OFFICE_IP/32

# SSH
aws ec2 authorize-security-group-ingress --group-id $LGTM_SG \
  --protocol tcp --port 22 --cidr YOUR_OFFICE_IP/32
```

#### Security Group: `sg-laravel-app`

Applied to all **6 Laravel EC2 instances**. No inbound rules needed from the LGTM server (Alloy pushes, not pulls).

| Port | Protocol | Source             | Purpose            |
| ---- | -------- | ------------------ | ------------------ |
| 80   | TCP      | ALB / 0.0.0.0/0    | HTTP traffic       |
| 443  | TCP      | ALB / 0.0.0.0/0    | HTTPS traffic      |
| 22   | TCP      | Your IP / VPN CIDR | SSH administration |

> **Key Insight**: Alloy uses a **push model** — it initiates outbound connections to the LGTM server. The Laravel security group only needs normal app ports; no special inbound rules for observability.

---

## 2. S3 Lifecycle Policy

This tiered policy reduces storage costs automatically:

| Age (Days) | Storage Class       | Cost Behavior                          |
| ---------- | ------------------- | -------------------------------------- |
| 0–30       | S3 Standard         | Hot — fast access, higher cost         |
| 30–100     | Intelligent-Tiering | Warm — auto-moves between access tiers |
| 100+       | **Deleted**         | Data permanently removed               |

Apply the lifecycle policy:

```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket YOUR_COMPANY-observability-data \
  --lifecycle-configuration file://aws/s3-lifecycle-policy.json
```

The policy file is at [`aws/s3-lifecycle-policy.json`](aws/s3-lifecycle-policy.json).

> **Why Intelligent-Tiering instead of Glacier?** Observability data older than 30 days is rarely accessed but occasionally needed for incident investigation. Intelligent-Tiering lets AWS optimize cost automatically without you managing retrieval delays.

---

## 3. LGTM Server EC2 Setup

### 3.1 Recommended Instance Sizing

| Metric                    | Recommendation                                     |
| ------------------------- | -------------------------------------------------- |
| **Instance Type**         | `t3.xlarge` (4 vCPU, 16 GB RAM) — minimum          |
| **Better for production** | `m6i.xlarge` (4 vCPU, 16 GB RAM) — consistent perf |
| **Root Volume**           | 100 GB gp3 (for WAL, local caches, Docker images)  |
| **OS**                    | Ubuntu 24.04 LTS or Amazon Linux 2023              |

> **Why this size?** Running Loki, Mimir, Tempo, and Grafana in single-binary mode on one host requires ~8-12 GB of RAM under normal load from 6 nodes. The `t3.xlarge` is cost-effective to start; upgrade to `m6i.xlarge` if you see CPU throttling.

### 3.2 Server Bootstrap

SSH into the LGTM EC2 instance and run:

```bash
# Update the system
sudo apt update && sudo apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER

# Install Docker Compose plugin
sudo apt install -y docker-compose-plugin

# Verify
docker --version
docker compose version

# Log out and back in for group membership to take effect
exit
```

### 3.3 Deploy the LGTM Stack

```bash
# Clone or copy this repo to the server
cd /opt
sudo mkdir -p lgtm && sudo chown $USER:$USER lgtm
git clone <your-repo-url> lgtm
# Or: scp -r lgtm-server/ user@lgtm-server:/opt/lgtm/

cd /opt/lgtm/lgtm-server

# IMPORTANT: Edit each config file to set the correct:
# - S3 bucket name
# - AWS region (endpoint)
# Then start the stack:
docker compose up -d

# Watch the logs to ensure everything starts cleanly
docker compose logs -f --tail=50
```

### 3.4 Configuration Files

All config files are in the `lgtm-server/` directory:

| File                                      | Component | Purpose                                |
| ----------------------------------------- | --------- | -------------------------------------- |
| `docker-compose.yml`                      | All       | Service definitions and networking     |
| `loki-config.yaml`                        | Loki      | Log ingestion + S3 storage + retention |
| `mimir-config.yaml`                       | Mimir     | Metrics storage + S3 + limits          |
| `tempo-config.yaml`                       | Tempo     | Trace storage + S3 + metrics generator |
| `alertmanager-fallback-config.yaml`       | Mimir     | Required fallback AlertManager config  |
| `grafana/provisioning/datasources/*.yaml` | Grafana   | Auto-configure datasources on boot     |

#### Key design decisions in the configs:

- **Single-binary mode**: Each component runs as a single process — simpler to operate with 6 nodes. Scale to microservices mode when you reach ~50+ nodes.
- **IAM-based S3 auth**: No access keys in config files; the EC2 Instance Profile provides credentials automatically.
- **Tempo metrics generator**: Automatically creates RED metrics (Rate, Errors, Duration) from traces and pushes them to Mimir, enabling service graphs in Grafana.
- **TSDB index for Loki**: The modern `tsdb` index type (replacing BoltDB) provides better performance and S3-native storage.

---

## 4. Grafana Alloy Setup (on each Laravel EC2)

### 4.1 Install Grafana Alloy

Run on **each of the 6 Laravel EC2 instances**:

```bash
# Add the Grafana APT repository
sudo apt install -y apt-transport-https software-properties-common
sudo mkdir -p /etc/apt/keyrings/
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt update

# Install Alloy
sudo apt install -y alloy

# Verify installation
alloy --version
```

### 4.2 Enable Nginx & PHP-FPM Status Endpoints

Alloy's built-in exporters need these status pages to scrape metrics from. **No separate exporter binaries are installed.**

#### Nginx `stub_status` (required for Nginx metrics)

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

#### PHP-FPM Status Page (required for PHP-FPM metrics)

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

> **What about Node Exporter?** You don't need it at all. Alloy's built-in `prometheus.exporter.unix` reads directly from Linux's `/proc` and `/sys` filesystems — same data, zero extra processes.

### 4.3 Deploy the Alloy Configuration

```bash
# Copy the Alloy config to the correct location
sudo cp alloy/config.alloy /etc/alloy/config.alloy

# CRITICAL: Edit the config and replace these placeholders:
#   - LGTM_SERVER_PRIVATE_IP → your LGTM server's private IP (e.g., 10.0.1.50)
#   - instance = "laravel-app-1" → unique name per EC2 (laravel-app-1 through laravel-app-6)
sudo nano /etc/alloy/config.alloy

# Restart Alloy to apply
sudo systemctl restart alloy
sudo systemctl status alloy

# Check Alloy logs for errors
sudo journalctl -u alloy -f --no-pager -n 50
```

### 4.4 What Alloy Collects (No Extra Installation Needed)

Alloy's configuration file handles everything with zero standalone exporters:

| What                          | Alloy Component               | Old Approach                       | Metrics                                                |
| ----------------------------- | ----------------------------- | ---------------------------------- | ------------------------------------------------------ |
| **System (CPU/RAM/Disk/Net)** | `prometheus.exporter.unix`    | Standalone Node Exporter binary    | `node_cpu_seconds_total`, `node_memory_*`, etc.        |
| **Nginx**                     | `prometheus.exporter.nginx`   | Standalone Nginx Exporter binary   | `nginx_connections_*`, `nginx_http_requests_total`     |
| **PHP-FPM**                   | `prometheus.exporter.php_fpm` | Standalone PHP-FPM Exporter binary | `phpfpm_active_processes`, `phpfpm_listen_queue`, etc. |
| **Laravel Logs**              | `loki.source.file`            | Same ✓                             | Log lines → Loki                                       |
| **Nginx/PHP-FPM Logs**        | `loki.source.file`            | Same ✓                             | Log lines → Loki                                       |
| **Traces (OTLP)**             | `otelcol.receiver.otlp`       | Same ✓                             | Spans → Tempo                                          |

> **Result**: Instead of managing 4 systemd services (Alloy + 3 exporters), you manage **just 1** (Alloy). Less maintenance, fewer failure points, and unified configuration.

---

## 5. Laravel Application Integration

This is where the **gold standard** approach really shines — full OpenTelemetry tracing gives you request-level visibility that no exporter can provide.

### 5.1 Install OpenTelemetry (Tracing)

On each Laravel EC2, in the Laravel project directory:

```bash
# Core OpenTelemetry SDK + OTLP exporter
composer require open-telemetry/sdk \
  open-telemetry/exporter-otlp \
  open-telemetry/transport-grpc

# Laravel auto-instrumentation (recommended)
composer require keepsuit/laravel-opentelemetry

# Publish the config
php artisan vendor:publish --provider="Keepsuit\LaravelOpenTelemetry\LaravelOpenTelemetryServiceProvider"
```

> **Why `keepsuit/laravel-opentelemetry`?** It auto-instruments HTTP requests, Eloquent queries, Redis calls, Queue jobs, and Artisan commands with **zero code changes**. This shows you exactly which database query or API call is slowing down a specific request — far more powerful than a simple PHP-FPM exporter.

### 5.2 Configure OpenTelemetry Environment

Add these to your Laravel `.env` file (reference: [`laravel/.env.otel.example`](laravel/.env.otel.example)):

```env
OTEL_SERVICE_NAME=laravel-app-1
OTEL_TRACES_EXPORTER=otlp
OTEL_METRICS_EXPORTER=otlp
OTEL_LOGS_EXPORTER=none
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf
OTEL_PHP_AUTOLOAD_ENABLED=true
OTEL_RESOURCE_ATTRIBUTES=deployment.environment=production,service.namespace=laravel
OTEL_TRACES_SAMPLER=parentbased_traceidratio
OTEL_TRACES_SAMPLER_ARG=1.0
```

| Variable                      | What it does                                                 |
| ----------------------------- | ------------------------------------------------------------ |
| `OTEL_SERVICE_NAME`           | Identifies this instance in Tempo (change per EC2)           |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | Points to Alloy's local OTLP receiver                        |
| `OTEL_LOGS_EXPORTER=none`     | Logs go via Alloy file tailing, not OTLP                     |
| `OTEL_TRACES_SAMPLER_ARG`     | `1.0` = 100% of requests traced. Lower for high-traffic apps |

### 5.3 Install TraceId Middleware (Log ↔ Trace Correlation)

This middleware injects the trace ID into every log entry, enabling one-click navigation from a log line in Loki to its full trace waterfall in Tempo.

```bash
# Copy the reference middleware
cp laravel/TraceIdMiddleware.php /var/www/html/app/Http/Middleware/TraceIdMiddleware.php
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

### 5.4 What You Get from OpenTelemetry

After setup, Grafana's **Explore → Tempo** shows full request waterfalls:

```
HTTP GET /api/orders  [245ms]
├─ middleware.handle   [2ms]
├─ eloquent.query      [180ms]  ← "SELECT * FROM orders WHERE..."
│   └─ db.connect      [5ms]
├─ redis.get           [8ms]   ← Cache lookup
└─ http.response       [1ms]
```

This is **dramatically more useful** than a PHP-FPM exporter metric like "5 active processes" — you can see exactly _why_ a request is slow.

### 5.5 (Optional) Application-Level Metrics

For business metrics (e.g., "orders processed per minute", "failed payments"), expose a Prometheus endpoint:

```bash
composer require promphp/prometheus_client_php
```

Or push metrics via OTLP — Alloy's OTLP receiver already accepts metrics and forwards them to Mimir.

> See [`laravel/README.md`](laravel/README.md) for the complete Laravel integration guide.

---

## 6. Grafana Configuration

### 6.1 Datasources

Datasources are **auto-provisioned** when Grafana starts. The provisioning file at `grafana/provisioning/datasources/datasources.yaml` configures:

| Datasource | Type       | URL                            | Features                           |
| ---------- | ---------- | ------------------------------ | ---------------------------------- |
| **Mimir**  | Prometheus | `http://mimir:9009/prometheus` | Default, exemplar → Tempo linking  |
| **Loki**   | Loki       | `http://loki:3100`             | Derived field → Tempo trace lookup |
| **Tempo**  | Tempo      | `http://tempo:3200`            | Service map, trace → log/metrics   |

### 6.2 Cross-Signal Correlation

The provisioning config enables powerful cross-correlation:

```
Logs (Loki) ←──── traceId ────→ Traces (Tempo)
                                      │
                                      ▼
                              Metrics (Mimir)
                           (via service graph)
```

- **Loki → Tempo**: Click a trace ID in any log line to jump to the full trace.
- **Tempo → Loki**: From any trace span, filter Loki logs by trace ID.
- **Tempo → Mimir**: Service graph topology and RED metrics auto-generated.

### 6.3 Recommended Dashboards

Import these community dashboards via **Grafana UI → Dashboards → Import**:

| Dashboard                    | Grafana ID | What it shows                                |
| ---------------------------- | ---------- | -------------------------------------------- |
| Node Exporter Full           | `1860`     | CPU, memory, disk, network per host          |
| Loki Dashboard (logs volume) | `13639`    | Log ingestion rate, error rates, top sources |
| Mimir / Prometheus Overview  | `3662`     | Metric ingestion, query performance          |

> **Note on dashboard compatibility**: Alloy's `prometheus.exporter.unix` emits the same `node_*` metrics as standalone Node Exporter, so dashboard `1860` works without changes. Nginx and PHP-FPM metric names are also compatible.

#### Custom Laravel Dashboard (create manually)

| Panel Title                 | Query Type | Query                                                                                                    |
| --------------------------- | ---------- | -------------------------------------------------------------------------------------------------------- |
| Error Rate (5xx)            | Mimir      | `rate(nginx_http_requests_total{status=~"5.."}[5m])`                                                     |
| Request Rate                | Mimir      | `rate(nginx_http_requests_total[5m])`                                                                    |
| PHP-FPM Active Processes    | Mimir      | `phpfpm_active_processes`                                                                                |
| PHP-FPM Queue Length        | Mimir      | `phpfpm_listen_queue`                                                                                    |
| CPU Usage per Host          | Mimir      | `100 - (avg by(instance)(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)`                          |
| Memory Usage per Host       | Mimir      | `(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100`                                |
| Disk Usage per Host         | Mimir      | `100 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"} * 100)` |
| Recent Errors (Log Panel)   | Loki       | `{job="laravel", level="ERROR"}`                                                                         |
| Laravel Log Volume by Level | Loki       | `sum by(level)(rate({job="laravel"}[5m]))`                                                               |
| Service Graph               | Tempo      | Use the built-in Service Graph / Node Graph panel                                                        |
| Request Latency (p95)       | Mimir      | `histogram_quantile(0.95, sum(rate(traces_spanmetrics_latency_bucket[5m])) by (le, service))`            |
| Error Rate by Service       | Mimir      | `sum(rate(traces_spanmetrics_calls_total{status_code="STATUS_CODE_ERROR"}[5m])) by (service)`            |

> The last two panels use **Tempo's metrics generator** — it automatically creates RED metrics from traces and pushes them to Mimir. No extra config needed.

---

## 7. Security & Networking

### 7.1 Network Topology

All communication is over the **AWS VPC private network**. No observability traffic traverses the public internet.

```
Laravel EC2 (10.0.1.10)  ──┐
Laravel EC2 (10.0.1.11)  ──┤
Laravel EC2 (10.0.1.12)  ──┤     Private Network (VPC)
Laravel EC2 (10.0.1.13)  ──┼──────────────────────────▶  LGTM EC2 (10.0.1.50)
Laravel EC2 (10.0.1.14)  ──┤
Laravel EC2 (10.0.1.15)  ──┘
```

- **Alloy → Loki**: HTTP push to `http://10.0.1.50:3100`
- **Alloy → Mimir**: HTTP push to `http://10.0.1.50:9009`
- **Alloy → Tempo**: OTLP HTTP push to `http://10.0.1.50:4318`

### 7.2 DNS vs Load Balancer

| Option                              | When to use                                        |
| ----------------------------------- | -------------------------------------------------- |
| **Private IP** (recommended)        | Single LGTM server, simplest setup                 |
| **Route 53 Private Hosted Zone**    | Nicer than IP; use `lgtm.internal.yourcompany.com` |
| **ALB (Application Load Balancer)** | Only if scaling to multiple LGTM servers later     |

**Recommended approach**: Use a **Route 53 private hosted zone** so you can change the LGTM server IP without updating every Alloy config.

```bash
# Create a private hosted zone
aws route53 create-hosted-zone \
  --name internal.yourcompany.com \
  --caller-reference $(date +%s) \
  --hosted-zone-config PrivateZone=true \
  --vpc VPCRegion=ap-southeast-1,VPCId=vpc-xxxxxxxx

# Create an A record for the LGTM server
aws route53 change-resource-record-sets \
  --hosted-zone-id ZXXXXXXXXXXXXX \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "lgtm.internal.yourcompany.com",
        "Type": "A",
        "TTL": 60,
        "ResourceRecords": [{"Value": "10.0.1.50"}]
      }
    }]
  }'
```

Then in Alloy configs, use `lgtm.internal.yourcompany.com` instead of the IP.

### 7.3 TLS / Authentication

For a **private VPC** deployment with security groups:

| Layer         | Recommendation                                                             |
| ------------- | -------------------------------------------------------------------------- |
| **Transport** | TLS is **optional** if all traffic stays within a private VPC              |
| **Auth**      | Currently disabled (`auth_enabled: false` / `multitenancy_enabled: false`) |
| **Grafana**   | Enable HTTPS via an ALB with ACM certificate for UI access                 |
| **Future**    | If you need multi-tenancy, enable tenant headers in Alloy and each backend |

#### If you want TLS for Alloy → LGTM (defense-in-depth):

1. Generate a self-signed CA and server cert using `openssl` or use AWS ACM Private CA.
2. Configure each Tempo/Loki/Mimir to serve TLS.
3. Configure Alloy endpoints with `tls { ca_file = "/path/to/ca.crt" }`.

> For most single-VPC deployments with strict security groups, TLS between internal services is not required but you may add it for compliance.

---

## 8. Verification & Testing

### 8.1 Verify LGTM Stack Health

After `docker compose up -d`, check each component:

```bash
# Check all containers are healthy
docker compose ps

# Expected output: all services should show "healthy"
# NAME      STATUS         PORTS
# grafana   Up (healthy)   0.0.0.0:3000->3000/tcp
# loki      Up (healthy)   0.0.0.0:3100->3100/tcp
# mimir     Up (healthy)   0.0.0.0:9009->9009/tcp
# tempo     Up (healthy)   0.0.0.0:3200->3200/tcp, 0.0.0.0:4317-4318->4317-4318/tcp

# Test Loki readiness
curl -s http://localhost:3100/ready
# Expected: "ready"

# Test Mimir readiness
curl -s http://localhost:9009/ready
# Expected: "ready"

# Test Tempo readiness
curl -s http://localhost:3200/ready
# Expected: "ready"

# Test Grafana
curl -s http://localhost:3000/api/health
# Expected: {"commit":"...","database":"ok","version":"..."}
```

### 8.2 Verify Alloy on Laravel EC2s

```bash
# Check Alloy service status
sudo systemctl status alloy

# Check for errors in the log
sudo journalctl -u alloy --since "5 minutes ago" --no-pager

# Alloy's built-in debug UI (accessible locally)
curl -s http://localhost:12345/ready
# Expected: "ready"
```

### 8.3 Verify Data Flow

#### Logs (Loki)

```bash
# From the LGTM server — query recent logs
curl -s "http://localhost:3100/loki/api/v1/query?query={job=%22laravel%22}&limit=5" | jq .

# Or use Grafana: navigate to Explore → Loki → query: {job="laravel"}
```

#### Metrics (Mimir)

```bash
# From the LGTM server — query a common metric
curl -s "http://localhost:9009/prometheus/api/v1/query?query=up" | jq .

# Expected: results showing your scraped targets
```

#### Traces (Tempo)

```bash
# Search for recent traces via Tempo API
curl -s "http://localhost:3200/api/search?limit=5" | jq .

# Or use Grafana: navigate to Explore → Tempo → Search
```

### 8.4 End-to-End Smoke Test

Run this from any Laravel EC2 to generate test data:

```bash
# Generate a test log entry
echo "[$(date '+%Y-%m-%d %H:%M:%S')] production.ERROR: Smoke test from $(hostname)" \
  >> /var/www/html/storage/logs/laravel.log

# Generate a test trace (via OTLP HTTP directly)
curl -X POST http://localhost:4318/v1/traces \
  -H "Content-Type: application/json" \
  -d '{
    "resourceSpans": [{
      "resource": {
        "attributes": [{"key": "service.name", "value": {"stringValue": "smoke-test"}}]
      },
      "scopeSpans": [{
        "scope": {"name": "test"},
        "spans": [{
          "traceId": "'"$(openssl rand -hex 16)"'",
          "spanId": "'"$(openssl rand -hex 8)"'",
          "name": "smoke-test-span",
          "kind": 1,
          "startTimeUnixNano": "'"$(date +%s)000000000"'",
          "endTimeUnixNano": "'"$(( $(date +%s) + 1 ))000000000"'",
          "status": {"code": 1}
        }]
      }]
    }]
  }'

# Wait 30 seconds, then verify in Grafana:
# 1. Explore → Loki → {job="laravel", level="ERROR"} → should see your smoke test
# 2. Explore → Tempo → Search → should see "smoke-test-span"
# 3. Explore → Mimir → query "up" → should see all 6 instances
```

### 8.5 Basic Alerting Setup

Create alert rules in Grafana for common scenarios:

#### Alert: High Error Rate (5xx responses)

1. Go to **Grafana → Alerting → Alert Rules → New Alert Rule**
2. **Query** (Mimir): `sum(rate(nginx_http_requests_total{status=~"5.."}[5m])) by (instance) > 0.5`
3. **Condition**: Is above `0.5` (more than 0.5 errors per second)
4. **Evaluation**: Every 1m, for 5m
5. **Labels**: `severity = critical`
6. **Notifications**: Configure a contact point (email, Slack, PagerDuty)

#### Alert: Instance Down

1. **Query** (Mimir): `up == 0`
2. **Condition**: Is equal to `0`
3. **Evaluation**: Every 1m, for 3m
4. **Labels**: `severity = critical`

#### Alert: Disk Usage > 85%

1. **Query** (Mimir): `100 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"} * 100) > 85`
2. **Condition**: Is above `85`
3. **Evaluation**: Every 5m, for 10m
4. **Labels**: `severity = warning`

#### Alert: High Request Latency (from Traces)

1. **Query** (Mimir): `histogram_quantile(0.95, sum(rate(traces_spanmetrics_latency_bucket[5m])) by (le)) > 2`
2. **Condition**: p95 latency above 2 seconds
3. **Evaluation**: Every 1m, for 5m
4. **Labels**: `severity = warning`

> This alert uses Tempo's metrics generator — it automatically creates latency histograms from trace data. No PHP-FPM exporter needed.

#### Alert: High Laravel Error Logs

1. **Query** (Loki): `sum(rate({job="laravel", level="ERROR"}[5m])) > 1`
2. **Condition**: Is above `1` (more than 1 error log per second)
3. **Evaluation**: Every 1m, for 5m
4. **Labels**: `severity = warning`

---

## File Structure

```
log-monitoring/
├── README.md                          ← This file (you are here)
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
│   └── config.alloy                   ← Alloy config (built-in exporters, no standalone binaries)
└── laravel/
    ├── README.md                      ← Laravel integration guide
    ├── TraceIdMiddleware.php           ← Reference: log↔trace correlation middleware
    └── .env.otel.example              ← Reference: OpenTelemetry .env variables
```

---

## Quick Reference — Post-Deployment Checklist

- [ ] S3 bucket created with encryption and public access blocked
- [ ] Lifecycle policy applied (30d Standard → Intelligent-Tiering → 100d delete)
- [ ] IAM Instance Profile attached to LGTM EC2
- [ ] Security groups configured (Loki:3100, Mimir:9009, Tempo:4317/4318, Grafana:3000)
- [ ] Docker and Docker Compose installed on LGTM EC2
- [ ] `docker compose up -d` — all services healthy
- [ ] Grafana accessible at `http://<LGTM-IP>:3000`
- [ ] Alloy installed on all 6 Laravel EC2 instances
- [ ] Nginx `stub_status` enabled on each Laravel EC2
- [ ] PHP-FPM `pm.status_path` enabled on each Laravel EC2
- [ ] Alloy config updated with correct LGTM server IP and unique instance names
- [ ] Alloy service running on all 6 instances (single service — no standalone exporters)
- [ ] OpenTelemetry packages installed in Laravel (`keepsuit/laravel-opentelemetry`)
- [ ] `.env` updated with OTEL variables on each instance
- [ ] TraceId middleware registered in Laravel
- [ ] Logs visible in Grafana → Explore → Loki
- [ ] Metrics visible in Grafana → Explore → Mimir
- [ ] Traces visible in Grafana → Explore → Tempo (with request waterfalls)
- [ ] Log ↔ Trace correlation working (click traceId in log → opens trace)
- [ ] Alert rules created for critical scenarios
- [ ] Grafana admin password changed from default
