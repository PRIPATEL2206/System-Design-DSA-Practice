# DevOps Phase 9 — Monitoring & Logging

> **Learning Goal:** Master observability — understand how to monitor systems, collect and analyze logs, set up alerting, and build dashboards that give complete visibility into production applications.

---

## Table of Contents

1. [Why Monitoring & Observability?](#1-why-monitoring--observability)
2. [The Three Pillars of Observability](#2-the-three-pillars-of-observability)
3. [Prometheus — Metrics Collection](#3-prometheus--metrics-collection)
4. [Grafana — Visualization & Dashboards](#4-grafana--visualization--dashboards)
5. [ELK Stack — Centralized Logging](#5-elk-stack--centralized-logging)
6. [Loki — Lightweight Log Aggregation](#6-loki--lightweight-log-aggregation)
7. [Alerting — Detection & Response](#7-alerting--detection--response)
8. [Distributed Tracing](#8-distributed-tracing)
9. [Monitoring Kubernetes](#9-monitoring-kubernetes)
10. [SLOs, SLIs, and Error Budgets](#10-slos-slis-and-error-budgets)
11. [Production Monitoring Patterns](#11-production-monitoring-patterns)
12. [Interview Mastery](#12-interview-mastery)

---

## 1. Why Monitoring & Observability?

### Beginner Explanation

Your application is running in production. Users are accessing it 24/7. How do you know:
- Is it working right now?
- Is it slow?
- Is it about to break?
- What caused the outage at 3am?
- Which of the 50 microservices is the bottleneck?

Without monitoring, you find out your app is broken **when users complain** (or your boss's boss calls). With monitoring, you find out **before users notice**, often fixing issues before they become outages.

### Monitoring vs Observability

```
MONITORING (traditional):
  "Is the system UP or DOWN?"
  "Is CPU above 90%?"
  Pre-defined checks → pre-defined answers
  Works well for KNOWN failure modes

OBSERVABILITY (modern):
  "WHY is the system slow for users in Europe but not the US?"
  "WHICH request caused the spike in latency?"
  Ability to ask arbitrary questions about system behavior
  Works for UNKNOWN failure modes (novel problems)

OBSERVABILITY = Monitoring + the ability to debug UNKNOWN problems
  using metrics, logs, and traces together
```

### The Cost of Not Monitoring

```
SCENARIO: E-commerce site, no monitoring

Friday 6pm: Deploy new version (looks fine in staging)
Friday 8pm: Checkout starts failing for 10% of users
Friday 10pm: Customer tweets "your site is broken"
Friday 11pm: Engineer finds tweet, starts investigating
Saturday 1am: Root cause found (memory leak in new code)
Saturday 2am: Rollback deployed, site recovers

Impact: 8 HOURS of degraded service
  → Lost revenue
  → Customer trust damaged
  → Engineer's weekend ruined

WITH MONITORING:

Friday 6pm: Deploy new version
Friday 6:05pm: Alert fires — "Error rate increased from 0.1% to 5%"
Friday 6:10pm: On-call engineer sees alert, checks dashboard
Friday 6:15pm: Identifies new deployment as cause, rolls back
Friday 6:20pm: Service restored

Impact: 20 MINUTES of degraded service
  → Minimal revenue loss
  → Users barely noticed
  → Post-mortem on Monday
```

---

## 2. The Three Pillars of Observability

```
┌─────────────────────────────────────────────────────────────────────┐
│                    THREE PILLARS OF OBSERVABILITY                    │
│                                                                     │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐     │
│  │     METRICS     │  │      LOGS       │  │     TRACES      │     │
│  │                 │  │                 │  │                 │     │
│  │  "What is       │  │  "What          │  │  "How does a    │     │
│  │   happening?"   │  │   happened?"    │  │   request flow  │     │
│  │                 │  │                 │  │   through the   │     │
│  │  Numbers over   │  │  Text records   │  │   system?"      │     │
│  │  time           │  │  of events      │  │                 │     │
│  │                 │  │                 │  │  Request path    │     │
│  │  CPU: 72%       │  │  "Connection    │  │  across services │     │
│  │  Latency: 45ms  │  │   refused to    │  │  with timing    │     │
│  │  Errors: 12/min │  │   database"     │  │                 │     │
│  │  Requests: 5k/s │  │                 │  │  API → Auth →   │     │
│  │                 │  │  Structured,    │  │  DB → Cache →   │     │
│  │  Prometheus     │  │  searchable     │  │  Response       │     │
│  │  Datadog        │  │                 │  │                 │     │
│  │  CloudWatch     │  │  ELK, Loki      │  │  Jaeger, Tempo  │     │
│  │                 │  │  CloudWatch     │  │  Zipkin, X-Ray  │     │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘     │
│                                                                     │
│  USE TOGETHER:                                                      │
│  1. Metric alert fires (error rate spiked)                         │
│  2. Check dashboard (which endpoint? which service?)               │
│  3. Search logs (find the actual error messages)                   │
│  4. Trace the failing request (find where it broke)                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### When to Use Each

| Pillar | Answers | Example |
|--------|---------|---------|
| **Metrics** | Is something wrong? How much? How fast? | "Error rate is 5%" |
| **Logs** | What specifically went wrong? | "NullPointerException in PaymentService.java:142" |
| **Traces** | Where in the system did it break? How long did each part take? | "Request spent 3s in database query, 100ms everywhere else" |

---

## 3. Prometheus — Metrics Collection

### What is Prometheus?

Prometheus is an open-source **time-series database and monitoring system**. It collects numerical metrics from your applications and infrastructure, stores them, and lets you query and alert on them.

**Founded:** 2012 at SoundCloud, donated to CNCF  
**Model:** Pull-based (Prometheus scrapes targets, not push)  
**Storage:** Built-in time-series database (TSDB)  
**Query language:** PromQL

### Prometheus Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    PROMETHEUS ECOSYSTEM                              │
│                                                                     │
│  TARGETS (things being monitored):                                  │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────────┐       │
│  │ App      │ │ Node     │ │ kube-    │ │ Custom App       │       │
│  │ /metrics │ │ Exporter │ │ state-   │ │ (instrumented)   │       │
│  │          │ │ /metrics │ │ metrics  │ │ /metrics         │       │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────────────┘       │
│       │             │            │             │                     │
│       └─────────────┴────────────┴─────────────┘                    │
│                             │                                       │
│                      SCRAPE (pull every 15s)                        │
│                             │                                       │
│                             ▼                                       │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    PROMETHEUS SERVER                          │   │
│  │                                                              │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │   │
│  │  │ Scrape       │  │ TSDB         │  │ Rule Engine      │   │   │
│  │  │ Manager      │  │ (time-series │  │ (recording &     │   │   │
│  │  │              │  │  storage)    │  │  alerting rules) │   │   │
│  │  └──────────────┘  └──────────────┘  └────────┬─────────┘   │   │
│  │                                               │              │   │
│  └───────────────────────────────────────────────┼──────────────┘   │
│                             │                    │                   │
│              ┌──────────────┘                    │                   │
│              ▼                                   ▼                   │
│  ┌─────────────────────┐            ┌─────────────────────┐         │
│  │      GRAFANA        │            │   ALERTMANAGER      │         │
│  │  (visualization)    │            │  (routes alerts to  │         │
│  │                     │            │   Slack, PagerDuty, │         │
│  │  Dashboards,        │            │   email)            │         │
│  │  queries, panels    │            │                     │         │
│  └─────────────────────┘            └─────────────────────┘         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Pull vs Push Model

```
PULL (Prometheus):
  Prometheus reaches out to each target and scrapes /metrics endpoint
  
  Advantages:
    ✅ Central control over what's monitored
    ✅ Easy to tell if a target is down (scrape fails)
    ✅ Targets don't need to know about Prometheus
    ✅ No data lost if Prometheus restarts (scrapes resume)
  
  Disadvantages:
    ❌ Doesn't work well for short-lived jobs (use Pushgateway)
    ❌ Targets must be network-reachable from Prometheus

PUSH (StatsD, Datadog Agent, CloudWatch):
  Applications push metrics to a central collector
  
  Advantages:
    ✅ Works for short-lived jobs/functions
    ✅ Works behind firewalls (outbound only)
  
  Disadvantages:
    ❌ Harder to detect down services (absence of data)
    ❌ Can overwhelm collector under load
```

### Metric Types

```
FOUR METRIC TYPES IN PROMETHEUS:

1. COUNTER — always increases (resets to 0 on restart)
   Use for: Total requests, total errors, bytes transferred
   Example: http_requests_total{method="GET", status="200"} = 1547892
   
   Query: rate(http_requests_total[5m]) = requests per second

2. GAUGE — goes up and down
   Use for: Temperature, memory usage, active connections, queue size
   Example: node_memory_available_bytes = 4294967296
   
   Query: node_memory_available_bytes = current value

3. HISTOGRAM — distribution of values (request durations, response sizes)
   Puts values into buckets (≤10ms, ≤50ms, ≤100ms, ≤500ms, ≤1s)
   Use for: Latency percentiles, request size distribution
   Example: http_request_duration_seconds_bucket{le="0.1"} = 24054
   
   Query: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
          = 99th percentile latency

4. SUMMARY — similar to histogram but calculates percentiles client-side
   Pre-calculated quantiles (p50, p90, p99)
   Less flexible than histograms (can't aggregate across instances)
```

### Instrumenting Your Application

```python
# Python application with prometheus_client
from prometheus_client import Counter, Histogram, Gauge, start_http_server
import time

# Define metrics
REQUEST_COUNT = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)

REQUEST_LATENCY = Histogram(
    'http_request_duration_seconds',
    'HTTP request latency',
    ['method', 'endpoint'],
    buckets=[0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0]
)

ACTIVE_REQUESTS = Gauge(
    'http_active_requests',
    'Number of active HTTP requests'
)

# Use in your request handler
def handle_request(method, endpoint):
    ACTIVE_REQUESTS.inc()
    start_time = time.time()
    
    try:
        response = process_request()
        status = "200"
    except Exception:
        status = "500"
    finally:
        duration = time.time() - start_time
        REQUEST_COUNT.labels(method=method, endpoint=endpoint, status=status).inc()
        REQUEST_LATENCY.labels(method=method, endpoint=endpoint).observe(duration)
        ACTIVE_REQUESTS.dec()

# Expose /metrics endpoint on port 8000
start_http_server(8000)
```

```go
// Go application with prometheus/client_golang
package main

import (
    "github.com/prometheus/client_golang/prometheus"
    "github.com/prometheus/client_golang/prometheus/promhttp"
    "net/http"
)

var (
    requestsTotal = prometheus.NewCounterVec(
        prometheus.CounterOpts{
            Name: "http_requests_total",
            Help: "Total HTTP requests",
        },
        []string{"method", "endpoint", "status"},
    )
    
    requestDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{
            Name:    "http_request_duration_seconds",
            Help:    "HTTP request latency",
            Buckets: prometheus.DefBuckets,
        },
        []string{"method", "endpoint"},
    )
)

func init() {
    prometheus.MustRegister(requestsTotal, requestDuration)
}

func main() {
    http.Handle("/metrics", promhttp.Handler())
    http.ListenAndServe(":8080", nil)
}
```

### Prometheus Configuration

```yaml
# prometheus.yml
global:
  scrape_interval: 15s        # How often to scrape targets
  evaluation_interval: 15s    # How often to evaluate rules
  scrape_timeout: 10s

# Alerting configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

# Load alerting/recording rules
rule_files:
  - "alerts/*.yml"
  - "recording_rules/*.yml"

# Scrape configurations (what to monitor)
scrape_configs:
  # Monitor Prometheus itself
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  # Monitor application pods (Kubernetes service discovery)
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      # Only scrape pods with annotation prometheus.io/scrape: "true"
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      # Use custom port from annotation
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
        action: replace
        target_label: __address__
        regex: (.+)
        replacement: ${1}:${2}

  # Monitor nodes (via node-exporter)
  - job_name: 'node-exporter'
    kubernetes_sd_configs:
      - role: node
    relabel_configs:
      - action: replace
        target_label: __address__
        replacement: ${1}:9100

  # Monitor external services
  - job_name: 'external-api'
    metrics_path: /metrics
    static_configs:
      - targets: ['api.example.com:9090']
```

### PromQL — Querying Metrics

```promql
# ─── BASIC QUERIES ──────────────────────────────────────────────

# Current value of a gauge
node_memory_available_bytes

# Filter by labels
http_requests_total{method="POST", status="500"}

# Regex matching
http_requests_total{status=~"5.."}   # All 5xx errors

# ─── RATE & INCREASE ───────────────────────────────────────────

# Requests per second (last 5 minutes)
rate(http_requests_total[5m])

# Requests per second for 500 errors
rate(http_requests_total{status="500"}[5m])

# Total increase over 1 hour
increase(http_requests_total[1h])

# ─── AGGREGATION ────────────────────────────────────────────────

# Total requests per second across all instances
sum(rate(http_requests_total[5m]))

# Requests per second grouped by status code
sum by (status) (rate(http_requests_total[5m]))

# Average latency across all instances
avg(rate(http_request_duration_seconds_sum[5m]) / rate(http_request_duration_seconds_count[5m]))

# ─── PERCENTILES (from histograms) ─────────────────────────────

# 99th percentile latency
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))

# 95th percentile latency per endpoint
histogram_quantile(0.95, sum by (le, endpoint) (rate(http_request_duration_seconds_bucket[5m])))

# ─── USEFUL PATTERNS ───────────────────────────────────────────

# Error rate (% of requests that are errors)
sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) * 100

# Availability (% of requests that succeed)
1 - (sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])))

# Memory usage percentage
(1 - node_memory_AvailableBytes / node_memory_MemTotal_bytes) * 100

# Disk will be full in X hours (prediction)
predict_linear(node_filesystem_avail_bytes[6h], 24*3600) < 0
```

---

## 4. Grafana — Visualization & Dashboards

### What is Grafana?

Grafana is an open-source **visualization and analytics platform**. It connects to data sources (Prometheus, Loki, Elasticsearch, CloudWatch) and displays data as dashboards with panels.

### Dashboard Structure

```
GRAFANA DASHBOARD:
┌─────────────────────────────────────────────────────────────────────┐
│  Dashboard: Production Overview                                     │
│  ┌───────────────────┐ ┌───────────────────┐ ┌───────────────────┐  │
│  │  Request Rate     │ │  Error Rate       │ │  Latency P99      │  │
│  │  ████████████     │ │  █                │ │  ████████         │  │
│  │  ████████████     │ │  █                │ │  ████████████     │  │
│  │  12,450 req/s     │ │  0.3%             │ │  142ms            │  │
│  └───────────────────┘ └───────────────────┘ └───────────────────┘  │
│  ┌─────────────────────────────────────────────────────────────────┐│
│  │  Request Latency (histogram heatmap)                            ││
│  │  ░░░░░░░░░░████████████████████████░░░░░░░░░░░░░░░░░░░░░░░░░  ││
│  │  ░░░░░░░░░░████████████████████████████░░░░░░░░░░░░░░░░░░░░░  ││
│  │  ░░░░░░░░░░████████████████████████████████████░░░░░░░░░░░░░  ││
│  └─────────────────────────────────────────────────────────────────┘│
│  ┌───────────────────────────────┐ ┌───────────────────────────────┐│
│  │  CPU Usage by Pod             │ │  Memory Usage by Pod          ││
│  │  pod-1: ████████ 62%         │ │  pod-1: ██████ 45%           ││
│  │  pod-2: ██████ 48%           │ │  pod-2: █████████ 72%        ││
│  │  pod-3: █████ 38%            │ │  pod-3: ██████ 51%           ││
│  └───────────────────────────────┘ └───────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────┘
```

### The Four Golden Signals

Google's SRE book defines four key metrics to monitor for every service:

```
┌─────────────────────────────────────────────────────────────────────┐
│                   FOUR GOLDEN SIGNALS                                │
│                                                                     │
│  1. LATENCY                                                         │
│     How long requests take to serve                                │
│     Track: p50, p90, p95, p99                                      │
│     Alert on: p99 > 500ms                                          │
│     Query: histogram_quantile(0.99, ...)                           │
│                                                                     │
│  2. TRAFFIC                                                         │
│     How much demand is being placed on the system                  │
│     Track: requests per second, concurrent users                   │
│     Alert on: sudden drop (means users can't reach you)            │
│     Query: sum(rate(http_requests_total[5m]))                      │
│                                                                     │
│  3. ERRORS                                                          │
│     Rate of requests that fail                                     │
│     Track: error rate %, errors per second                         │
│     Alert on: error rate > 1%                                      │
│     Query: rate(http_requests_total{status=~"5.."}[5m])            │
│                                                                     │
│  4. SATURATION                                                      │
│     How "full" your service is                                     │
│     Track: CPU %, memory %, disk %, queue depth                    │
│     Alert on: resources approaching capacity                       │
│     Query: container_memory_usage_bytes / container_spec_memory_limit│
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### RED Method (Request-focused)

```
For request-driven services (APIs, web servers):

R — Rate:      Requests per second
E — Errors:    Number of failed requests per second  
D — Duration:  Distribution of response times (histograms)

Every service dashboard should show these three.
```

### USE Method (Resource-focused)

```
For infrastructure resources (CPU, memory, disk, network):

U — Utilization:  % of resource being used (CPU at 72%)
S — Saturation:   Amount of work queued/waiting (run queue length)
E — Errors:       Count of error events (disk errors, network drops)

Every node/infrastructure dashboard should show these.
```

### Grafana Dashboard as Code

```json
{
  "dashboard": {
    "title": "API Service",
    "panels": [
      {
        "title": "Request Rate",
        "type": "graph",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{service=\"api\"}[5m]))",
            "legendFormat": "Total RPS"
          }
        ]
      },
      {
        "title": "Error Rate",
        "type": "stat",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total{service=\"api\",status=~\"5..\"}[5m])) / sum(rate(http_requests_total{service=\"api\"}[5m])) * 100",
            "legendFormat": "Error %"
          }
        ],
        "thresholds": {
          "steps": [
            {"color": "green", "value": 0},
            {"color": "yellow", "value": 1},
            {"color": "red", "value": 5}
          ]
        }
      }
    ]
  }
}
```

---

## 5. ELK Stack — Centralized Logging

### What is ELK?

ELK is a stack of three tools for centralized log management:

| Component | Role |
|-----------|------|
| **Elasticsearch** | Search engine and storage (indexes and stores logs) |
| **Logstash** | Log processing pipeline (parse, transform, route logs) |
| **Kibana** | Web UI for searching and visualizing logs |

Modern variant: **EFK** (Fluentd replaces Logstash — lighter, more Kubernetes-native)

### ELK Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                      ELK STACK ARCHITECTURE                         │
│                                                                     │
│  SOURCES:                                                           │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────────┐                           │
│  │App 1 │ │App 2 │ │App 3 │ │Syslog    │                           │
│  │stdout │ │stdout │ │stdout │ │/var/log  │                           │
│  └──┬───┘ └──┬───┘ └──┬───┘ └────┬─────┘                           │
│     │        │        │          │                                  │
│     └────────┴────────┴──────────┘                                  │
│                    │                                                │
│                    ▼                                                │
│  ┌──────────────────────────────────────┐                           │
│  │  BEATS / FLUENTD / LOGSTASH          │  ← Collection layer      │
│  │  (Ship logs to central location)     │                           │
│  │                                      │                           │
│  │  Filebeat: reads log files           │                           │
│  │  Fluentd: K8s-native log collector   │                           │
│  └──────────────────┬───────────────────┘                           │
│                     │                                               │
│                     ▼                                               │
│  ┌──────────────────────────────────────┐                           │
│  │  LOGSTASH (optional)                 │  ← Processing layer      │
│  │  (Parse, transform, enrich, filter)  │                           │
│  │                                      │                           │
│  │  - Parse JSON logs                   │                           │
│  │  - Extract fields from unstructured  │                           │
│  │  - Add geo-IP data                   │                           │
│  │  - Redact sensitive fields           │                           │
│  └──────────────────┬───────────────────┘                           │
│                     │                                               │
│                     ▼                                               │
│  ┌──────────────────────────────────────┐                           │
│  │  ELASTICSEARCH                       │  ← Storage + Search      │
│  │  (Index, store, full-text search)    │                           │
│  │                                      │                           │
│  │  Cluster: 3+ nodes for HA           │                           │
│  │  Indexes: one per day (logs-2024.01) │                           │
│  │  Retention: 30-90 days typically    │                           │
│  └──────────────────┬───────────────────┘                           │
│                     │                                               │
│                     ▼                                               │
│  ┌──────────────────────────────────────┐                           │
│  │  KIBANA                              │  ← Visualization         │
│  │  (Search, filter, dashboards)        │                           │
│  │                                      │                           │
│  │  - Full-text search across all logs  │                           │
│  │  - Filter by service, time, level    │                           │
│  │  - Create visualizations             │                           │
│  │  - Anomaly detection (ML)            │                           │
│  └──────────────────────────────────────┘                           │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Structured Logging (Critical Best Practice)

```
❌ UNSTRUCTURED LOGS (hard to search and parse):
  2024-01-15 14:32:01 INFO User john logged in from 10.0.1.5
  2024-01-15 14:32:02 ERROR Failed to process payment for order 12345: timeout

✅ STRUCTURED LOGS (JSON — machine-parseable):
  {"timestamp":"2024-01-15T14:32:01Z","level":"info","message":"User logged in","user":"john","ip":"10.0.1.5","service":"auth"}
  {"timestamp":"2024-01-15T14:32:02Z","level":"error","message":"Payment failed","order_id":"12345","reason":"timeout","service":"payment"}

WHY STRUCTURED:
  - Searchable: find all logs where order_id=12345
  - Filterable: show only level=error AND service=payment
  - Aggregatable: count errors per service per minute
  - Parseable: no regex gymnastics needed
```

```python
# Python structured logging with structlog
import structlog

logger = structlog.get_logger()

logger.info("payment_processed",
    order_id="12345",
    amount=99.99,
    currency="USD",
    user_id="user-abc",
    duration_ms=142
)

# Output:
# {"event": "payment_processed", "order_id": "12345", "amount": 99.99, 
#  "currency": "USD", "user_id": "user-abc", "duration_ms": 142,
#  "timestamp": "2024-01-15T14:32:01Z", "level": "info"}
```

### Logstash Pipeline Configuration

```ruby
# logstash.conf
input {
  beats {
    port => 5044    # Receive from Filebeat/Fluentd
  }
}

filter {
  # Parse JSON logs
  json {
    source => "message"
  }
  
  # Add geo-IP data from client IP
  geoip {
    source => "client_ip"
  }
  
  # Redact sensitive data
  mutate {
    gsub => [
      "message", "\b\d{16}\b", "[REDACTED_CC]",     # Credit card numbers
      "password", ".*", "[REDACTED]"
    ]
  }
  
  # Parse timestamps
  date {
    match => ["timestamp", "ISO8601"]
    target => "@timestamp"
  }
}

output {
  elasticsearch {
    hosts => ["elasticsearch:9200"]
    index => "logs-%{[service]}-%{+YYYY.MM.dd}"
  }
}
```

### ELK Pros and Cons

| Pros | Cons |
|------|------|
| Powerful full-text search | Resource-heavy (Elasticsearch needs lots of RAM) |
| Rich visualization (Kibana) | Complex to operate at scale |
| Mature ecosystem | Expensive (storage + compute) |
| Flexible schema | JVM-based (memory hungry) |
| ML-powered anomaly detection | Index management overhead |

---

## 6. Loki — Lightweight Log Aggregation

### What is Loki?

Loki is a **horizontally-scalable log aggregation system** by Grafana Labs. It's designed to be cost-effective and easy to operate, using the same labels approach as Prometheus.

**Key difference from ELK:** Loki does NOT index log content. It only indexes labels (metadata). This makes it 10-100x cheaper to operate but means full-text search across log content is slower.

### Loki vs ELK

```
ELK (Elasticsearch):
  Indexes EVERY WORD in every log line
  → Fast full-text search: "find all logs containing 'timeout'"
  → Expensive: indexes are large, need lots of RAM and disk
  → Complex: cluster management, shard rebalancing, mapping conflicts

LOKI:
  Indexes ONLY labels (service, pod, namespace, level)
  Stores log content as compressed chunks (like object storage)
  → Search by labels is instant: {service="api", level="error"}
  → Grep within results: {service="api"} |= "timeout"
  → Cheap: just compressed text storage (S3/GCS)
  → Simple: no cluster management, works like Prometheus

WHEN TO USE WHICH:
  Loki: Cost-sensitive, Kubernetes-native, Grafana stack, label-based filtering is sufficient
  ELK:  Need fast full-text search across ALL logs, compliance/audit requirements, complex analytics
```

### Loki Architecture

```
┌──────────────────────────────────────────────────────────┐
│                  LOKI ARCHITECTURE                        │
│                                                          │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐         │
│  │  Promtail  │  │  Promtail  │  │  Promtail  │         │
│  │  (agent)   │  │  (agent)   │  │  (agent)   │         │
│  │  on each   │  │  on each   │  │  on each   │         │
│  │  node      │  │  node      │  │  node      │         │
│  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘         │
│        │               │               │                 │
│        └───────────────┴───────────────┘                 │
│                        │                                 │
│                   Push logs                              │
│                        │                                 │
│                        ▼                                 │
│  ┌────────────────────────────────────────────────────┐  │
│  │                    LOKI                            │  │
│  │                                                    │  │
│  │  Label Index: {service="api", pod="api-abc123"}    │  │
│  │  Log Chunks: stored in S3/GCS (compressed)        │  │
│  │                                                    │  │
│  └────────────────────────────────────────────────────┘  │
│                        │                                 │
│                        ▼                                 │
│  ┌────────────────────────────────────────────────────┐  │
│  │                  GRAFANA                           │  │
│  │  LogQL queries, log panel in dashboards           │  │
│  └────────────────────────────────────────────────────┘  │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

### LogQL (Loki Query Language)

```logql
# ─── LABEL FILTERING (indexed, fast) ────────────────────────

# All logs from the api service
{service="api"}

# Errors from api service in production namespace
{service="api", namespace="production", level="error"}

# Regex on labels
{service=~"api|payment", level="error"}

# ─── LINE FILTERING (grep over content) ─────────────────────

# Logs containing "timeout"
{service="api"} |= "timeout"

# Logs NOT containing "healthcheck"
{service="api"} != "healthcheck"

# Regex on log content
{service="api"} |~ "status=(4|5)\\d{2}"

# ─── PARSING AND FILTERING ──────────────────────────────────

# Parse JSON and filter on fields
{service="api"} | json | status >= 500

# Parse logfmt and filter
{service="api"} | logfmt | duration > 1s

# ─── AGGREGATIONS (for metrics from logs) ───────────────────

# Count error logs per minute
count_over_time({service="api", level="error"}[1m])

# Rate of logs per second
rate({service="api"}[5m])

# Top 5 services by error count
topk(5, count_over_time({level="error"}[1h]))

# Bytes of logs per service
sum by (service) (bytes_over_time({namespace="production"}[1h]))
```

### Promtail Configuration (Kubernetes)

```yaml
# Promtail DaemonSet config (runs on each node, ships pod logs to Loki)
apiVersion: v1
kind: ConfigMap
metadata:
  name: promtail-config
data:
  promtail.yaml: |
    server:
      http_listen_port: 9080
    
    positions:
      filename: /tmp/positions.yaml
    
    clients:
      - url: http://loki:3100/loki/api/v1/push
    
    scrape_configs:
      - job_name: kubernetes-pods
        kubernetes_sd_configs:
          - role: pod
        pipeline_stages:
          - docker: {}
          - json:
              expressions:
                level: level
                msg: message
          - labels:
              level:
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_label_app]
            target_label: service
          - source_labels: [__meta_kubernetes_namespace]
            target_label: namespace
          - source_labels: [__meta_kubernetes_pod_name]
            target_label: pod
```

---

## 7. Alerting — Detection & Response

### Alerting Philosophy

```
GOOD ALERTS:
  ✅ Actionable — someone needs to DO something
  ✅ Meaningful — indicates real user impact
  ✅ Timely — fires soon enough to respond
  ✅ Relevant — routes to the right person/team

BAD ALERTS (alert fatigue):
  ❌ Non-actionable — "CPU at 75%" (so what?)
  ❌ Noisy — flapping alerts, auto-resolving in 30 seconds
  ❌ Too late — fires after users already noticed
  ❌ No runbook — alert fires, nobody knows what to do
  
RULE: Every alert should either wake someone up or shouldn't exist.
      If you wouldn't wake someone up for it, it's a dashboard metric, not an alert.
```

### Alert Severity Levels

| Severity | Response | Example | Notification |
|----------|---------|---------|-------------|
| **Critical (P1)** | Immediate (page on-call) | Site down, data loss | PagerDuty, phone call |
| **High (P2)** | Respond within 30 min | Degraded performance, partial outage | Slack + PagerDuty |
| **Warning (P3)** | Respond within hours | Disk filling up, approaching limits | Slack channel |
| **Info (P4)** | Review next business day | Build failed, cert expiring in 30 days | Email |

### Prometheus Alerting Rules

```yaml
# alerts/application.yml
groups:
  - name: application-alerts
    rules:
      # ─── ERROR RATE ──────────────────────────────────────────
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) 
          / sum(rate(http_requests_total[5m])) > 0.05
        for: 5m    # Must be true for 5 minutes before firing
        labels:
          severity: critical
          team: platform
        annotations:
          summary: "High error rate: {{ $value | humanizePercentage }}"
          description: "Error rate is above 5% for 5 minutes"
          runbook_url: "https://wiki.example.com/runbooks/high-error-rate"
          dashboard_url: "https://grafana.example.com/d/api-overview"

      # ─── LATENCY ─────────────────────────────────────────────
      - alert: HighLatencyP99
        expr: |
          histogram_quantile(0.99, 
            sum by (le) (rate(http_request_duration_seconds_bucket[5m]))
          ) > 2.0
        for: 10m
        labels:
          severity: high
          team: platform
        annotations:
          summary: "P99 latency is {{ $value }}s (threshold: 2s)"
          runbook_url: "https://wiki.example.com/runbooks/high-latency"

      # ─── SATURATION ──────────────────────────────────────────
      - alert: PodMemoryHigh
        expr: |
          container_memory_usage_bytes / container_spec_memory_limit_bytes > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Pod {{ $labels.pod }} memory at {{ $value | humanizePercentage }}"

      # ─── AVAILABILITY ────────────────────────────────────────
      - alert: ServiceDown
        expr: up{job="api"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Service {{ $labels.instance }} is DOWN"

  - name: infrastructure-alerts
    rules:
      # ─── DISK SPACE ──────────────────────────────────────────
      - alert: DiskSpaceLow
        expr: |
          (node_filesystem_avail_bytes / node_filesystem_size_bytes) < 0.15
        for: 15m
        labels:
          severity: warning
        annotations:
          summary: "Node {{ $labels.instance }} disk at {{ $value | humanizePercentage }} free"

      # ─── PREDICTIVE ALERT ────────────────────────────────────
      - alert: DiskWillFillIn24h
        expr: |
          predict_linear(node_filesystem_avail_bytes[6h], 24*3600) < 0
        for: 30m
        labels:
          severity: high
        annotations:
          summary: "Node {{ $labels.instance }} disk predicted to fill within 24 hours"
```

### Alertmanager Configuration

```yaml
# alertmanager.yml
global:
  resolve_timeout: 5m
  slack_api_url: 'https://hooks.slack.com/services/xxx/yyy/zzz'

# Route tree — how alerts are routed to receivers
route:
  receiver: 'default-slack'        # Default receiver
  group_by: ['alertname', 'service']
  group_wait: 30s                  # Wait before sending first notification
  group_interval: 5m               # Wait between notifications for same group
  repeat_interval: 4h              # Re-send if still firing after this
  
  routes:
    # Critical alerts → PagerDuty (wake someone up)
    - match:
        severity: critical
      receiver: 'pagerduty-critical'
      repeat_interval: 15m
    
    # High alerts → Slack #incidents channel
    - match:
        severity: high
      receiver: 'slack-incidents'
      repeat_interval: 1h
    
    # Team-specific routing
    - match:
        team: database
      receiver: 'slack-database-team'

# Inhibition rules — suppress alerts when a higher-severity alert is firing
inhibit_rules:
  - source_match:
      severity: critical
    target_match:
      severity: warning
    equal: ['service']    # Same service — don't send warning if critical is firing

# Receivers — where alerts go
receivers:
  - name: 'default-slack'
    slack_configs:
      - channel: '#monitoring'
        title: '{{ .GroupLabels.alertname }}'
        text: '{{ range .Alerts }}{{ .Annotations.summary }}{{ end }}'
  
  - name: 'pagerduty-critical'
    pagerduty_configs:
      - service_key: 'your-pagerduty-service-key'
        severity: critical
  
  - name: 'slack-incidents'
    slack_configs:
      - channel: '#incidents'
        color: '{{ if eq .Status "firing" }}danger{{ else }}good{{ end }}'
        title: '{{ .CommonAnnotations.summary }}'
        text: |
          *Alert:* {{ .CommonLabels.alertname }}
          *Severity:* {{ .CommonLabels.severity }}
          *Description:* {{ .CommonAnnotations.description }}
          *Runbook:* {{ .CommonAnnotations.runbook_url }}
```

### On-Call Best Practices

```
ON-CALL STRUCTURE:

1. PRIMARY ON-CALL
   - First responder to pages
   - Acknowledges within 5 minutes
   - Escalates if can't resolve in 30 minutes

2. SECONDARY ON-CALL
   - Backup if primary doesn't respond
   - Auto-escalated after 10 minutes

3. ESCALATION POLICY:
   0 min:   Page primary on-call
   10 min:  Page secondary on-call
   30 min:  Page engineering manager
   60 min:  Page VP Engineering

4. EVERY ALERT MUST HAVE:
   - Runbook URL (step-by-step resolution guide)
   - Dashboard URL (relevant Grafana dashboard)
   - Owner (which team is responsible)
   - Severity (determines response time)

5. POST-INCIDENT:
   - Blameless post-mortem within 48 hours
   - Action items to prevent recurrence
   - Update runbooks with new knowledge
```

---

## 8. Distributed Tracing

### What is Distributed Tracing?

In a microservices architecture, a single user request may pass through 5-20 services. When something is slow, which service is the bottleneck?

**Distributed tracing** tracks a request across all services, recording timing at each step.

```
USER REQUEST: GET /checkout

┌─────────────────────────────────────────────────────────────────┐
│  Trace ID: abc-123-def-456                                      │
│                                                                 │
│  ├── API Gateway (5ms)                                          │
│  │   ├── Auth Service (12ms)                                    │
│  │   │   └── Redis cache lookup (2ms)                           │
│  │   ├── Cart Service (8ms)                                     │
│  │   │   └── PostgreSQL query (5ms)                             │
│  │   ├── Payment Service (2,350ms) ← BOTTLENECK!               │
│  │   │   ├── Stripe API call (2,300ms) ← External dependency   │
│  │   │   └── PostgreSQL write (45ms)                            │
│  │   └── Notification Service (15ms)                            │
│  │       └── Kafka produce (3ms)                                │
│  │                                                              │
│  Total: 2,410ms                                                 │
│  Without tracing: "checkout is slow" (but which part?)          │
│  With tracing: "Stripe API is taking 2.3s — their issue or ours?"│
└─────────────────────────────────────────────────────────────────┘
```

### Tracing Concepts

| Term | Meaning |
|------|---------|
| **Trace** | The entire journey of a request through the system |
| **Span** | A single operation within a trace (one service call) |
| **Trace ID** | Unique identifier propagated across all services |
| **Span ID** | Unique identifier for each individual operation |
| **Parent Span** | The span that initiated this span |
| **Context Propagation** | Passing trace/span IDs in HTTP headers between services |

### OpenTelemetry (The Standard)

```python
# Python application with OpenTelemetry
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.flask import FlaskInstrumentor
from opentelemetry.instrumentation.requests import RequestsInstrumentor

# Set up tracing
provider = TracerProvider()
processor = BatchSpanProcessor(OTLPSpanExporter(endpoint="http://tempo:4317"))
provider.add_span_processor(processor)
trace.set_tracer_provider(provider)

# Auto-instrument Flask and HTTP requests
FlaskInstrumentor().instrument_app(app)
RequestsInstrumentor().instrument()

# Custom spans for business logic
tracer = trace.get_tracer(__name__)

@app.route("/checkout")
def checkout():
    with tracer.start_as_current_span("process_checkout") as span:
        span.set_attribute("user_id", current_user.id)
        span.set_attribute("cart_items", len(cart))
        
        # Each of these creates a child span automatically
        validate_cart()
        payment = process_payment(cart.total)
        send_confirmation(payment)
        
        return {"status": "success"}
```

### Tracing Tools

| Tool | Type | Best For |
|------|------|----------|
| **Jaeger** | Open source (CNCF) | Self-hosted, production-proven |
| **Tempo** | Open source (Grafana Labs) | Grafana stack, object storage backend |
| **Zipkin** | Open source | Simple, mature |
| **AWS X-Ray** | Managed (AWS) | AWS-native applications |
| **Datadog APM** | SaaS | Full-stack observability |

---

## 9. Monitoring Kubernetes

### What to Monitor in Kubernetes

```
LAYER 1: CLUSTER HEALTH
  - Node status (Ready/NotReady)
  - Node resource usage (CPU, memory, disk)
  - etcd health and latency
  - API server response times
  - Scheduler queue depth

LAYER 2: WORKLOAD HEALTH
  - Pod status (Running, Pending, CrashLoopBackOff)
  - Pod restarts
  - Container resource usage vs limits
  - Deployment rollout status
  - HPA current vs target replicas

LAYER 3: APPLICATION HEALTH
  - Request rate, error rate, latency (RED)
  - Business metrics (signups, orders, revenue)
  - Dependency health (DB connections, external APIs)
  - Queue depths, processing times
```

### Kubernetes Monitoring Stack

```yaml
# Standard Kubernetes monitoring stack (kube-prometheus-stack Helm chart)
# Installs: Prometheus, Grafana, Alertmanager, node-exporter, kube-state-metrics

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack \
    --namespace monitoring \
    --create-namespace \
    --set grafana.adminPassword=admin \
    --set prometheus.prometheusSpec.retention=30d \
    --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=100Gi

# This installs:
# - Prometheus (metrics collection)
# - Grafana (dashboards — pre-configured K8s dashboards!)
# - Alertmanager (alert routing)
# - node-exporter (host metrics on every node)
# - kube-state-metrics (K8s object metrics — pod status, deployments, etc.)
# - Pre-built dashboards and alerts for K8s
```

### Key Kubernetes Metrics

```promql
# ─── POD HEALTH ────────────────────────────────────────────────

# Pods in CrashLoopBackOff
kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"} > 0

# Pod restarts (last hour)
increase(kube_pod_container_status_restarts_total[1h]) > 3

# Pods pending (can't be scheduled)
kube_pod_status_phase{phase="Pending"} > 0

# ─── RESOURCE USAGE ───────────────────────────────────────────

# Container CPU usage vs request
sum by (pod) (rate(container_cpu_usage_seconds_total[5m])) 
/ sum by (pod) (kube_pod_container_resource_requests{resource="cpu"})

# Container memory usage vs limit (OOM risk)
container_memory_usage_bytes / container_spec_memory_limit_bytes > 0.9

# ─── NODE HEALTH ───────────────────────────────────────────────

# Nodes not ready
kube_node_status_condition{condition="Ready", status="true"} == 0

# Node CPU usage
1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m]))

# Node memory usage
1 - (node_memory_AvailableBytes / node_memory_MemTotal_bytes)

# ─── DEPLOYMENT HEALTH ─────────────────────────────────────────

# Deployments with fewer replicas than desired
kube_deployment_status_replicas_available < kube_deployment_spec_replicas

# HPA at maximum (can't scale further)
kube_horizontalpodautoscaler_status_current_replicas 
== kube_horizontalpodautoscaler_spec_max_replicas
```

---

## 10. SLOs, SLIs, and Error Budgets

### Definitions

```
SLI (Service Level Indicator):
  A quantitative measure of service performance.
  "What are we measuring?"
  Example: "The proportion of successful HTTP requests" = 99.2%

SLO (Service Level Objective):
  A target value for an SLI.
  "What's our goal?"
  Example: "99.9% of requests should succeed over 30 days"

SLA (Service Level Agreement):
  A contract with consequences if SLO is missed.
  "What happens if we miss the goal?"
  Example: "If uptime < 99.95%, customer gets credits"

ERROR BUDGET:
  The allowed amount of unreliability.
  If SLO = 99.9% over 30 days:
    Error budget = 0.1% of 30 days = 43.2 minutes of downtime allowed
    
  When error budget is consumed:
    → Freeze feature deployments
    → Focus on reliability work
    → No new risks until budget recovers
```

### SLO Examples

| Service | SLI | SLO | Error Budget (30 days) |
|---------|-----|-----|----------------------|
| API | Successful requests (non-5xx) | 99.9% | 43.2 minutes |
| API | Latency p99 < 500ms | 99% | 7.2 hours |
| Checkout | Successful transactions | 99.95% | 21.6 minutes |
| Dashboard | Page load < 3s | 95% | 36 hours |

### Implementing SLOs with Prometheus

```yaml
# Recording rules — pre-calculate SLI values for dashboards
groups:
  - name: slo-recording-rules
    rules:
      # Request success rate (SLI)
      - record: sli:http_request_success_rate:ratio_rate5m
        expr: |
          sum(rate(http_requests_total{status!~"5.."}[5m]))
          / sum(rate(http_requests_total[5m]))

      # Latency SLI (% of requests faster than 500ms)
      - record: sli:http_request_latency_within_threshold:ratio_rate5m
        expr: |
          sum(rate(http_request_duration_seconds_bucket{le="0.5"}[5m]))
          / sum(rate(http_request_duration_seconds_count[5m]))

      # Error budget remaining (30-day window)
      - record: slo:error_budget_remaining:ratio
        expr: |
          1 - (
            (1 - sli:http_request_success_rate:ratio_rate30d) / (1 - 0.999)
          )

  - name: slo-alerts
    rules:
      # Alert when burning error budget too fast
      - alert: ErrorBudgetBurnRateHigh
        expr: |
          slo:error_budget_remaining:ratio < 0.5
        for: 5m
        labels:
          severity: high
        annotations:
          summary: "Error budget is more than 50% consumed"
          description: "At current error rate, SLO will be breached"
```

---

## 11. Production Monitoring Patterns

### Pattern 1: The Monitoring Stack

```
RECOMMENDED PRODUCTION STACK:

Metrics:   Prometheus + Thanos/Cortex (long-term storage)
Dashboards: Grafana
Logs:      Loki (or ELK if you need full-text search)
Traces:    Tempo (or Jaeger)
Alerts:    Alertmanager → PagerDuty + Slack
Uptime:    External healthcheck (UptimeRobot, Pingdom)

WHY THIS STACK:
  - Open source (no licensing costs)
  - Cloud-native (designed for Kubernetes)
  - Unified in Grafana (metrics + logs + traces in one UI)
  - Horizontally scalable
  - Active community
```

### Pattern 2: Dashboards Hierarchy

```
LEVEL 1: EXECUTIVE DASHBOARD (overview)
  - Overall availability (SLO status)
  - Error budget remaining
  - Revenue impact metrics
  - Active incidents

LEVEL 2: SERVICE DASHBOARD (per service)
  - Request rate, error rate, latency (RED)
  - Pod health, resource usage
  - Dependency health
  - Deployment markers (when did deploys happen?)

LEVEL 3: DEBUG DASHBOARD (detailed)
  - Per-endpoint metrics
  - Database query performance
  - Cache hit rates
  - Individual pod metrics
  - Logs correlation
```

### Pattern 3: Deployment Annotations

```promql
# Show deployment events on Grafana graphs
# When a spike correlates with a deployment marker → deployment caused it

# Annotate graphs with deployment events using Grafana annotations API:
# POST /api/annotations
{
  "time": 1704067200000,
  "text": "Deployed v2.1.0 (sha-abc123)",
  "tags": ["deployment", "api"]
}
```

### Pattern 4: Runbook-Driven Alerting

```
EVERY ALERT MUST LINK TO A RUNBOOK:

Runbook template:
─────────────────────────────────────────
# Alert: HighErrorRate

## What this means
The API is returning 5xx errors at a rate above 5%.
Users are experiencing failures.

## Impact
- Checkout failures
- Revenue loss estimated at $X/minute

## Investigation steps
1. Check recent deployments: `kubectl rollout history deployment/api`
2. Check error logs: {service="api", level="error"} in Grafana Explore
3. Check dependencies: Is the database responding? Is Redis up?
4. Check resource saturation: CPU throttling? Memory pressure?

## Resolution steps
- If caused by recent deployment: `kubectl rollout undo deployment/api`
- If database issue: Check RDS dashboard, failover if needed
- If traffic spike: Check if HPA is scaling, increase max replicas if needed

## Escalation
If not resolved in 15 minutes, escalate to database team / platform team.
─────────────────────────────────────────
```

---

## 12. Interview Mastery

---

### Beginner Interview Questions

---

**Q1: What is observability and how is it different from monitoring?**

**Perfect Answer:**

"Monitoring is checking pre-defined metrics against thresholds — 'is CPU above 90%?' or 'is the service returning 200s?' It answers known questions about known failure modes.

Observability is the ability to understand a system's internal state from its external outputs — metrics, logs, and traces. It lets you answer questions you didn't anticipate: 'Why are users in Germany experiencing higher latency than users in the US?' or 'Which specific database query is causing the checkout timeout?'

The three pillars of observability are:
1. **Metrics** — numeric time-series data (CPU usage, request rate, error rate)
2. **Logs** — discrete events with context (error messages, audit trails)
3. **Traces** — the path of a request through distributed services with timing

You use them together: a metric alert tells you SOMETHING is wrong, logs tell you WHAT went wrong, and traces tell you WHERE in the system it went wrong.

The practical difference: with monitoring alone, you can detect known problems. With observability, you can debug novel, unknown problems by correlating data across all three pillars."

---

**Q2: Explain the difference between Prometheus and ELK. When would you use each?**

**Perfect Answer:**

"Prometheus and ELK solve different problems:

**Prometheus** is a time-series database for **metrics** — numerical measurements over time. It answers 'how much' and 'how fast': CPU usage is 72%, error rate is 0.5%, p99 latency is 200ms. It scrapes targets every 15 seconds, stores efficiently, and enables alerting via PromQL.

**ELK (Elasticsearch + Logstash + Kibana)** is a log management system for **text data** — the actual content of log messages. It answers 'what happened': 'Connection refused to database at 14:32:01 for user john, order #12345.' It provides full-text search across all your logs.

**When to use Prometheus:**
- Dashboards showing system health over time
- Alerting on thresholds (error rate > 5%)
- Capacity planning (resource utilization trends)
- SLO tracking

**When to use ELK (or Loki):**
- Debugging a specific error (search for the exception message)
- Auditing (who did what, when)
- Investigating an incident (what happened in the 2 minutes before the crash)
- Compliance (retain all logs for N years)

In practice, you use BOTH: Prometheus alert fires → engineer opens Grafana → sees the error spike → pivots to log search to find the actual error message → finds the root cause.

A modern alternative to ELK is Loki — it's lighter and cheaper because it only indexes labels (not log content), which is sufficient for most Kubernetes use cases."

---

**Q3: What are the Four Golden Signals?**

**Perfect Answer:**

"The Four Golden Signals are from Google's SRE book — the most important metrics for any request-driven service:

1. **Latency** — how long requests take. Important to separate successful request latency from error latency (a 500 error in 5ms shouldn't lower your 'average latency'). Track percentiles: p50, p90, p99.

2. **Traffic** — how much demand is on the system. Measured in requests per second for web services, or I/O operations per second for storage. A sudden DROP in traffic is also an alert — it might mean users can't reach you.

3. **Errors** — the rate of failed requests. Either explicit (HTTP 5xx) or implicit (200 response but wrong content, or response took > 5s which violates the SLO).

4. **Saturation** — how 'full' the service is. CPU utilization, memory usage, disk I/O, thread pool usage. The key is to alert BEFORE hitting 100% — at 80-90% you're approaching degradation.

For my dashboards, every service's overview panel shows these four signals. If all four look normal, the service is healthy. If any deviate, that's where I start investigating.

There's also the RED method (Rate, Errors, Duration) which is a simplified version focused specifically on request-driven microservices."

---

### Intermediate Interview Questions

---

**Q4: How would you set up monitoring for a Kubernetes cluster running 20 microservices?**

**Perfect Answer:**

"I'd implement a layered monitoring approach covering infrastructure, platform, and application:

**Stack selection:**
- Metrics: Prometheus (via kube-prometheus-stack Helm chart)
- Dashboards: Grafana (bundled with kube-prometheus-stack)
- Logs: Loki + Promtail (lightweight, Kubernetes-native)
- Traces: Tempo + OpenTelemetry (correlates with metrics/logs in Grafana)
- Alerts: Alertmanager → PagerDuty for critical, Slack for warnings
- External uptime: separate external probe (Pingdom or custom health checker outside the cluster)

**Layer 1 — Infrastructure (comes free with kube-prometheus-stack):**
- Node metrics (node-exporter on each node): CPU, memory, disk, network
- Kubernetes metrics (kube-state-metrics): pod status, deployment health, HPA state
- Pre-built alerts: node down, disk filling, pods crashing

**Layer 2 — Platform services:**
- Ingress controller metrics (requests per second, error rates, latency)
- Database metrics (RDS CloudWatch or postgres-exporter)
- Cache metrics (redis-exporter)
- Message queue metrics (kafka-exporter)

**Layer 3 — Application (each of 20 services):**
- Instrument each service with a Prometheus client library
- Expose /metrics endpoint with RED metrics (rate, errors, duration)
- Add service-specific business metrics (orders processed, users created)
- OpenTelemetry for distributed tracing across services

**Dashboard hierarchy:**
1. Cluster overview (all services at a glance)
2. Per-service dashboards (auto-generated from labels)
3. Debug dashboards (per-endpoint, per-pod detail)

**Alerting strategy:**
- SLO-based alerts: 'error budget burning too fast'
- Symptom-based, not cause-based: alert on 'error rate high' not 'CPU high'
- Every alert has a runbook URL
- On-call rotation with PagerDuty escalation

**Key principle:** Monitor symptoms (user-visible impact) and use debugging tools (logs, traces) to find causes. Don't alert on causes directly — CPU at 90% might be fine if latency is still good."

---

**Q5: Explain how Prometheus scraping works. How does service discovery work in Kubernetes?**

**Perfect Answer:**

"Prometheus uses a pull model — it actively reaches out to targets and scrapes their /metrics endpoint at a configured interval (typically 15 seconds).

**Basic flow:**
1. Prometheus reads its configuration to know WHAT to scrape
2. At each scrape interval, it sends HTTP GET to each target's /metrics endpoint
3. Target responds with all current metric values in Prometheus exposition format
4. Prometheus stores the scraped data in its time-series database
5. If a scrape fails → target is marked 'down' (the `up` metric = 0)

**Kubernetes service discovery:**

In Kubernetes, pods come and go constantly. You can't hardcode target IPs. Prometheus integrates with the Kubernetes API to dynamically discover what to scrape.

Service discovery roles:
- `role: pod` — discovers all pods in the cluster
- `role: service` — discovers all services
- `role: endpoints` — discovers service endpoints
- `role: node` — discovers all nodes

The common pattern:
1. Configure Prometheus with `kubernetes_sd_configs` role: pod
2. Use `relabel_configs` to filter which pods to scrape
3. Convention: pods with annotation `prometheus.io/scrape: 'true'` get scraped
4. The annotation `prometheus.io/port: '8080'` tells Prometheus which port to use

```yaml
relabel_configs:
  - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
    action: keep
    regex: true
```

This means: any pod in the cluster that has the annotation automatically gets scraped — no manual configuration needed when new services are deployed. This is why Prometheus is so well-suited to Kubernetes: zero-config service discovery.

**Scaling challenge:** With 1000+ pods, a single Prometheus instance might struggle. Solutions: sharding with Thanos/Cortex, or Prometheus federation (hierarchical scraping)."

---

**Q6: What is alert fatigue and how do you prevent it?**

**Perfect Answer:**

"Alert fatigue occurs when teams receive so many alerts that they start ignoring all of them — including the critical ones. It's one of the most dangerous monitoring anti-patterns because it makes your alerting system worse than useless (people actively dismiss legitimate alerts).

**Common causes:**
- Alerting on every metric threshold instead of user-visible symptoms
- Flapping alerts (fires for 30 seconds, resolves, fires again)
- Too low thresholds (alerting at 60% CPU when nothing is actually wrong)
- No severity differentiation (everything pages with equal urgency)
- Alerting on things no one can action

**Prevention strategies:**

1. **Alert on symptoms, not causes:**
   - Bad: 'CPU > 80%' (maybe the service handles it fine)
   - Good: 'Error rate > 1% for 5 minutes' (users are affected)

2. **Use the `for` clause aggressively:**
   - Don't fire instantly — require the condition to persist (5-15 minutes)
   - Prevents one-blip alerts from waking people up

3. **Severity-based routing:**
   - Critical: pages someone (service is down, data at risk)
   - Warning: goes to Slack (something to investigate during business hours)
   - Info: goes to dashboard/email (FYI, no action needed)

4. **Regular alert review:**
   - Monthly: review which alerts fired, which were actionable
   - Delete or demote alerts that are never acted upon
   - If an alert fires and the response is 'acknowledge and ignore' → delete it

5. **SLO-based alerting:**
   - Instead of many threshold alerts, use error budget burn rate
   - One alert that says 'you're burning error budget too fast' replaces 10 granular alerts

6. **Inhibition and correlation:**
   - If the database is down, suppress all application-level alerts (they're caused by the DB)
   - Alertmanager's `inhibit_rules` handles this

The goal: if an alert fires, it always means 'a human needs to do something right now.' If not, it shouldn't be an alert — it should be a dashboard panel."

---

### Advanced Interview Questions

---

**Q7: Design a monitoring strategy that supports an SLO of 99.95% availability for a payment processing system.**

**Perfect Answer:**

"99.95% availability over 30 days = 21.6 minutes of allowed downtime per month. This is extremely tight — every minute of downtime consumes significant error budget. The monitoring system must detect issues in under 2 minutes and support instant decision-making.

**SLI Definition:**
- Primary SLI: proportion of payment API requests that return success (2xx) within 5 seconds
- Excludes: health checks, internal metrics scraping
- Measured at the load balancer (not inside the app — captures ALL failures)

**Multi-signal detection (under 2 minutes):**

1. **Real-time error rate monitoring:**
```promql
# Alert if error budget is being consumed at 14.4x normal rate
# (would exhaust monthly budget in 2 hours if sustained)
sum(rate(payment_requests_total{status!~"2.."}[5m]))
/ sum(rate(payment_requests_total[5m])) > 0.007  # 0.7% error rate
```

2. **Latency degradation:**
```promql
histogram_quantile(0.99, rate(payment_duration_seconds_bucket[5m])) > 5
```

3. **Synthetic monitoring (external probes):**
- External service makes test payments every 30 seconds from multiple regions
- If synthetic fails: alert immediately (doesn't wait for 5-minute `for` clause)
- Tests the full path: DNS → LB → service → database → payment provider

4. **Dependency monitoring:**
- Payment provider API latency and error rate
- Database connection pool utilization
- Redis availability

**Error budget tracking:**
```
Dashboard shows:
  - Error budget remaining: 18.2 minutes (84% remaining)
  - Current burn rate: 0.3x (normal)
  - Projected budget at month end: 15.1 minutes (healthy)
```

**Alert tiers:**

| Condition | Severity | Action |
|-----------|----------|--------|
| Budget burn >14x (exhausts in 2h) | Critical (page) | Immediate rollback |
| Budget burn >6x (exhausts in 12h) | High (page) | Investigate within 15min |
| Budget burn >3x (exhausts in 3 days) | Warning (Slack) | Investigate same day |
| Budget < 25% remaining | High | Feature freeze until recovered |

**Incident response automation:**
- If error rate spikes immediately after a deployment: auto-rollback (no human needed)
- If a specific payment provider is failing: auto-failover to backup provider
- Circuit breaker: if downstream is failing, fail fast with clear error instead of timing out

**Post-incident:**
- Every incident that consumed > 5 minutes of error budget gets a post-mortem
- Monthly SLO review with engineering leadership
- Error budget policy: when budget is exhausted, freeze deployments and focus on reliability

This approach means issues are detected in 1-2 minutes (scrape interval + for clause), response begins in < 5 minutes (on-call SLA), and many scenarios auto-remediate without human intervention."

---

**Q8: Your monitoring shows that P99 latency suddenly doubled but P50 is unchanged. What does this mean and how do you investigate?**

**Perfect Answer:**

"P99 doubled but P50 is unchanged tells me that the median user experience is fine, but a small percentage of requests (roughly the slowest 1%) are taking much longer. This is a 'long tail' latency problem.

**What this typically means:**
The issue affects a subset of requests — not all traffic. Something is slow for specific conditions.

**Investigation approach:**

**1. Identify WHICH requests are slow:**
```promql
# Break down P99 by endpoint
histogram_quantile(0.99, sum by (le, endpoint) (rate(http_request_duration_seconds_bucket[5m])))
```
Is it one endpoint or all? If one endpoint → that endpoint has a problem.

**2. Identify WHEN it started:**
Look at the time series — was it sudden (deployment?) or gradual (growing data/load)?
Check deployment markers on the dashboard.

**3. Common causes of P99 spike with stable P50:**

| Cause | How to Identify |
|-------|-----------------|
| **Garbage collection pauses** | GC metrics (jvm_gc_pause_seconds), correlated with latency spikes |
| **Database query plan regression** | One slow query for a specific data pattern (e.g., user with 10,000 orders) |
| **Resource contention (noisy neighbor)** | One pod is on a saturated node → check `kubectl top nodes` |
| **Connection pool exhaustion** | Pool is full, some requests wait for a connection. Check pool metrics |
| **DNS resolution timeouts** | Occasional DNS cache miss causes 5-second timeout |
| **Cold cache after restart** | First requests after pod restart hit database instead of cache |
| **External dependency slow for some regions** | Payment provider slow for EU requests but fine for US |
| **CPU throttling** | Pod hitting CPU limit → throttled → some requests delayed |

**4. Deep investigation:**
```bash
# Check if it correlates with specific pods
histogram_quantile(0.99, sum by (le, pod) (...))
# One pod much slower? → noisy neighbor, OOM pressure, or bad node

# Check distributed traces for slow requests
# Filter traces by duration > 2x normal
# Examine the span breakdown — which service/operation is slow
```

**5. CPU throttling check (very common hidden cause):**
```promql
rate(container_cpu_cfs_throttled_seconds_total[5m]) > 0
```
If CPU throttling is occurring: the pod hits its CPU limit, Linux CFS throttles it for the rest of the period, causing latency spikes for requests that land during the throttled interval. Fix: increase CPU limit or reduce CPU-intensive work.

**Key insight for interviews:** P99 problems are harder than P50 problems because they affect a minority of requests and can be caused by conditions that only occur sporadically. The investigation requires correlating metrics, traces, and logs to find the specific condition that triggers the slow path."

---

### Scenario-Based Questions

---

**Q9: You receive a page at 2am: 'Error rate above 5% for 5 minutes.' Walk through your incident response.**

**Perfect Answer:**

"This is a production incident. I follow a structured response:

**Minute 0-2: Acknowledge and assess**
- Acknowledge the page (stops escalation timer)
- Open the linked dashboard — what's the current error rate? Getting worse or stable?
- Quick check: is this a full outage or partial degradation?
- Post in incident channel: 'Investigating elevated error rate, currently at X%'

**Minute 2-5: Identify scope**
- Which service? (Check per-service error rate breakdown)
- Which endpoints? (Is it all traffic or specific paths?)
- Which errors? (Check logs: is it 500, 502, 503, 504?)
- When did it start? (Correlate with deployment markers on graph)

**Minute 5-10: Correlate with changes**
```bash
kubectl rollout history deployment/api -n production
```
- Was there a deployment in the last 30 minutes? If yes → strong candidate for rollback
- Any infrastructure changes? (Terraform apply, node scaling, config change)
- Any dependency failures? (Database, cache, external API)

**Decision point: Rollback or investigate?**
- If clearly correlated with a deployment: ROLLBACK IMMEDIATELY
  ```bash
  kubectl rollout undo deployment/api -n production
  ```
  Restore service first, root-cause after.
- If not deployment-related: continue investigating

**Minute 10-15: Deep investigation**
- Check logs for the error: `{service='api', level='error'} |= 'the error message'`
- Check traces: find failing requests, examine where they fail
- Check dependencies: is the database responding? Is Redis up? External APIs?
- Check resources: is a pod being OOM killed or CPU throttled?

**Minute 15-30: Resolution**
- Apply fix (rollback, restart, config change, scale up)
- Verify error rate is dropping back to normal
- Update incident channel with resolution

**After resolution:**
- Post in incident channel: 'Resolved at [time]. Error rate back to normal. Root cause: [brief]. Post-mortem to follow.'
- Write post-mortem within 48 hours (blameless — focus on what, not who)
- Create action items to prevent recurrence

**Key principles:**
1. Restore service first, investigate after (rollback is always safe)
2. Communicate continuously (stakeholders need to know what's happening)
3. Don't heroically debug for 30 minutes while users suffer — if you don't know the cause in 10 minutes, rollback
4. Every incident improves the system (post-mortem action items)"

---

**Q10: How would you reduce your Prometheus storage costs while retaining long-term metrics?**

**Perfect Answer:**

"Prometheus local storage is designed for short-term (15-30 days). For long-term retention, you need a tiered strategy:

**The problem:** Raw Prometheus data at 15-second scrape interval across 1000+ targets generates terabytes over months. Storing all of it in Prometheus is expensive and unnecessary.

**Solution: Tiered retention with downsampling**

**Tier 1: Prometheus local (0-15 days, full resolution)**
- Full 15-second resolution for recent data
- Used for real-time dashboards and alerting
- Stored on fast SSD (local to Prometheus)

**Tier 2: Long-term store (15 days - 2 years, downsampled)**
Use Thanos or Cortex to ship data to object storage (S3/GCS):

```
Thanos Architecture:
  Prometheus → Thanos Sidecar → uploads blocks to S3
  Thanos Compactor → downsamples: 5m resolution after 30 days, 1h after 90 days
  Thanos Store Gateway → serves queries from S3
  Thanos Query → unified query across Prometheus + S3
```

**Cost reduction techniques:**

1. **Downsampling:** After 30 days, keep 5-minute resolution instead of 15-second. After 90 days, keep 1-hour resolution. Storage drops 20x.

2. **Recording rules:** Pre-compute expensive queries and store only the result:
```yaml
# Instead of storing per-pod metrics for 1000 pods:
- record: job:http_requests_total:sum_rate5m
  expr: sum by (job) (rate(http_requests_total[5m]))
```
Store the aggregated result, not every individual series.

3. **Drop high-cardinality labels:** Labels like `request_id` or `user_id` create millions of time series. Use relabeling to drop them:
```yaml
metric_relabel_configs:
  - source_labels: [__name__]
    regex: 'expensive_metric_.*'
    action: drop
```

4. **Reduce scrape targets:** Do you really need 15-second resolution for every metric? Use different scrape intervals per job:
```yaml
- job_name: 'critical-services'
  scrape_interval: 15s
- job_name: 'batch-workers'
  scrape_interval: 60s     # Less critical, save 4x storage
```

5. **Object storage (cheapest tier):** S3 Standard → S3-IA for data older than 30 days. Thanos handles this transparently.

**Result:** In my experience, this approach reduces storage costs by 80-90% while retaining years of data for capacity planning and trend analysis. The key insight: you rarely need 15-second resolution for data older than 2 weeks. Downsampled data is perfectly adequate for long-term trends."

---

### Key Interview Terms

| Term | When to Use |
|------|------------|
| **Four Golden Signals** | Defining what to monitor per service |
| **RED method** | Request-driven service monitoring |
| **USE method** | Infrastructure/resource monitoring |
| **SLI / SLO / SLA** | Reliability targets and error budgets |
| **Error budget** | Balancing velocity vs reliability |
| **Alert fatigue** | Why too many alerts is dangerous |
| **PromQL** | Prometheus query examples |
| **Pull vs Push** | Prometheus scraping model |
| **Cardinality** | Why too many labels breaks Prometheus |
| **Service discovery** | How Prometheus finds Kubernetes targets |
| **Recording rules** | Pre-computing expensive queries |
| **Structured logging** | JSON logs with parseable fields |
| **Distributed tracing** | Request flow across services |
| **OpenTelemetry** | Industry standard for instrumentation |
| **Thanos / Cortex** | Long-term Prometheus storage |

---

[⬇️ Download This File](#)

---

*Phase 9 Complete. Ready for Phase 10 — Networking.*