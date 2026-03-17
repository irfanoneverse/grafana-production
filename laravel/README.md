# Laravel Integration — OpenTelemetry Tracing & Log Correlation

This directory contains reference files for integrating your Laravel application with the LGTM observability stack. This is the **application-side** setup — the infrastructure side is handled by Grafana Alloy.

## What This Enables

| Signal          | How                                                   | Result in Grafana                         |
| --------------- | ----------------------------------------------------- | ----------------------------------------- |
| **Traces**      | OpenTelemetry auto-instruments HTTP, DB, Redis, Queue | Full request waterfall in Tempo           |
| **Log ↔ Trace** | TraceId middleware injects trace ID into log context  | Click a log line in Loki → jumps to trace |
| **App Metrics** | (Optional) Prometheus counters for business events    | Custom dashboards in Mimir                |

---

## 1. Install OpenTelemetry Packages

```bash
# Core OpenTelemetry SDK + OTLP exporter
composer require open-telemetry/sdk \
  open-telemetry/exporter-otlp \
  open-telemetry/transport-grpc

# Laravel auto-instrumentation (recommended over manual setup)
composer require keepsuit/laravel-opentelemetry

# Publish the config file
php artisan vendor:publish --provider="Keepsuit\LaravelOpenTelemetry\LaravelOpenTelemetryServiceProvider"
```

> **Why `keepsuit/laravel-opentelemetry`?** It auto-instruments HTTP requests, Eloquent queries, Redis, Queue jobs, and Artisan commands with zero code changes. Much more powerful than manually wrapping spans.

## 2. Configure Environment Variables

Copy the variables from [`.env.otel.example`](.env.otel.example) into your Laravel `.env`:

```bash
# Adjust the path to your Laravel project directory
cat .env.otel.example >> /home/forge/southbound-api-qlmdtqda.on-forge.com/current/.env
```

Then edit `OTEL_SERVICE_NAME` to match the instance (e.g., `duadualive-staging`, `laravel-app-2`).

## 3. Install the TraceId Middleware

Copy [`TraceIdMiddleware.php`](TraceIdMiddleware.php) to your app:

```bash
# Adjust the path to your Laravel project directory
cp TraceIdMiddleware.php /home/forge/southbound-api-qlmdtqda.on-forge.com/current/app/Http/Middleware/TraceIdMiddleware.php
```

Register it in `bootstrap/app.php` (Laravel 11+):

```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->append(\App\Http\Middleware\TraceIdMiddleware::class);
})
```

Or for Laravel 10, add it to `$middleware` in `app/Http/Kernel.php`:

```php
protected $middleware = [
    // ... existing middleware
    \App\Http\Middleware\TraceIdMiddleware::class,
];
```

## 4. (Optional) Application-Level Metrics

For business metrics (e.g., orders processed, API calls counted), install a Prometheus client:

```bash
composer require promphp/prometheus_client_php
```

Expose a `/metrics` endpoint that Alloy can scrape, or push metrics via OTLP (already configured in Alloy's OTLP receiver).

## 5. Verify

After deploying, verify traces are flowing:

```bash
# Check Alloy is receiving OTLP data
curl -s http://localhost:12345/ready

# Generate a trace by hitting any Laravel route
curl -s http://localhost/api/health

# Check Tempo for traces (from LGTM server)
curl -s "http://LGTM_SERVER:3200/api/search?limit=5" | jq .
```

In Grafana: **Explore → Tempo → Search** — you should see spans from your Laravel app with database queries, HTTP calls, and queue jobs auto-instrumented.
