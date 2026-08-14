
# Monitoring & Observability – Quick Guide

## 1. Monitoring Overview

Monitoring helps track the **health, performance, and availability** of infrastructure, applications, databases, middleware, and logs.

Main monitoring areas:

| Layer | What We Monitor | Common Tool |
|---|---|---|
| Infrastructure | CPU, RAM, Disk, Network | Node Exporter |
| Application | Requests, Errors, Latency, Health | Prometheus / OpenTelemetry |
| Database | Connections, Queries, Storage, Locks | DB Exporter |
| Middleware | Redis, Kafka, RabbitMQ metrics | Service Exporters |
| Logs | Errors, Exceptions, Events | Loki |
| Traces | Request flow across services | OpenTelemetry |

---

## 2. Infrastructure Monitoring

**Node Exporter** is used for Linux server metrics.

Common metrics:

- CPU Usage
- Memory Usage
- Disk Usage
- Network Traffic
- Load Average
- System Uptime

Flow:

```text
Server → Node Exporter → Prometheus → Grafana
````

---

## 3. Application Monitoring

Application monitoring tracks:

* Request Rate
* Response Time / Latency
* Error Rate
* HTTP 4xx / 5xx
* Application Health
* Runtime metrics

Example:

```text
Application → Metrics → Prometheus → Grafana
```

---

## 4. Database & Middleware Monitoring

Database/middleware monitoring checks:

* Connections
* Queries / Commands
* Memory
* Storage
* Latency
* Errors
* Replication / Queue status

Examples:

```text
Redis       → Redis Exporter
MySQL       → MySQL Exporter
PostgreSQL  → PostgreSQL Exporter
Kafka       → Kafka Exporter
```

Example Redis metrics:

```text
Memory
Connected Clients
Keys
Commands
Evictions
Expired Keys
```

---

## 5. Log Monitoring

Logs contain detailed events such as:

```text
INFO
WARN
ERROR
```

**Loki** is commonly used for log collection and querying.

Flow:

```text
Application / Server
        ↓
      Loki
        ↓
    Grafana
```

LogQL is used to query logs.

Example:

```logql
{app="my-app"} |= "ERROR"
```

---

## 6. Distributed Tracing

Tracing follows a request across multiple services.

Example:

```text
Frontend
   ↓
API
   ↓
Redis
   ↓
Database
```

It helps identify **where the request is slow or failing**.

---

## 7. OpenTelemetry

**OpenTelemetry (OTel)** is used for collecting and instrumenting:

* Metrics
* Logs
* Traces

Example:

```text
Application
     ↓
OpenTelemetry
   ┌─┼─┐
   ↓ ↓ ↓
Metrics Logs Traces
```

It can send telemetry to different backends such as Prometheus, Loki, and Tempo/Jaeger.

---

## 8. Prometheus

Prometheus is mainly used for **metrics collection, storage, querying, and alert rule evaluation**.

It uses **PromQL**.

Example:

```promql
redis_up
```

---

## 9. Grafana

Grafana is used to **visualize metrics, logs, and traces** through dashboards.

```text
Prometheus ──┐
Loki ────────┼→ Grafana
Tracing ─────┘
```

---

## 10. Alerting

### Metric Alerts

Usually evaluated by Prometheus.

Example:

```promql
redis_connected_clients > 5000
```

Flow:

```text
Prometheus → Alert Rule → Alertmanager → Email/Slack
```

### Log Alerts

Can be created using LogQL-based conditions.

```text
Loki → LogQL Condition → Alerting → Notification
```

---

## 11. Quick Revision

```text
Node Exporter  → Server
App Metrics    → Application
DB Exporter    → Database
Redis Exporter → Redis
Loki           → Logs
OpenTelemetry  → Metrics + Logs + Traces
Prometheus     → Metrics
Grafana        → Visualization
Alertmanager   → Alert Routing / Notifications
```

### Easy Memory Trick

> **Metrics = Prometheus | Logs = Loki | Traces = OpenTelemetry | Dashboard = Grafana | Alerts = Alertmanager**

```
```
