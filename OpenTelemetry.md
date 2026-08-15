# OpenTelemetry – OTMS Monitoring Overview

## 1. What is OpenTelemetry?

OpenTelemetry (OTel) is an observability framework used to collect:

* **Traces** – request journey
* **Metrics** – application performance
* **Logs** – application events/errors

OTel itself is **not a database or dashboard**.

---

## 2. OTMS OpenTelemetry Flow

```text
Users
  ↓
AWS ALB
  ↓
Frontend
  ↓
OTel Instrumentation
  ↓
OTel-enabled APIs
  ├── Salary
  ├── Attendance
  ├── Employee
  └── Notification
        ↓
   OTel Collector
        ↓
 ┌──────┼──────┐
 ↓      ↓      ↓
Traces Metrics Logs
 ↓      ↓      ↓
Tempo Prometheus Loki
 └──────┼──────┘
        ↓
      Grafana
```

## 3. Main Components

### Instrumentation

Application ke andar telemetry generate karta hai.

### OTel Collector

Telemetry ko:

```text
Receive → Process → Export
```

karta hai.

### Backend

Telemetry ko store karta hai:

* **Traces → Tempo**
* **Metrics → Prometheus**
* **Logs → Loki**

### Grafana

Collected telemetry ko dashboards aur troubleshooting ke liye visualize karta hai.

---

## 4. Tracing Example

```text
User Request
     ↓
Frontend
     ↓
Salary API
     ↓
Redis
     ↓
ScyllaDB
```

Trace se pata chalega ki request **exactly kis service/database par slow hui**.

Example:

```text
Total Request = 970ms

Frontend  = 50ms
Salary    = 100ms
Redis     = 20ms
ScyllaDB  = 800ms  ← Issue
```

---

## 5. OTMS Implementation Plan

```text
1. Deploy OTel Collector
        ↓
2. Instrument Salary API
        ↓
3. Send telemetry using OTLP
        ↓
4. Configure Trace Backend
        ↓
5. Connect Grafana
        ↓
6. Add Metrics
        ↓
7. Add Logs
        ↓
8. Instrument remaining APIs
        ↓
9. Create Dashboards & Alerts
```

### Recommended Approach

Pehle sirf **Salary API** ka end-to-end POC karo:

```text
Salary API
    ↓
OpenTelemetry
    ↓
OTel Collector
    ↓
Tempo
    ↓
Grafana
```

Successful hone ke baad same architecture **Attendance, Employee aur Notification APIs** par implement karo.


                 OTMS
                   |
      ┌────────────┼────────────┐
      ↓            ↓            ↓
    Infra       Database      Redis
      |            |            |
 Node Exporter  DB Exporter  Redis Exporter
      |            |            |
      └────────────┼────────────┘
                   ↓
               Prometheus
                   ↓
                Grafana
                   ↓
              Alertmanager


              APPLICATIONS
                   |
            OpenTelemetry
                   |
            OTel Collector
              /         \
             ↓           ↓
          Metrics      Traces
             |           |
        Prometheus      Tempo
             \           /
              \         /
                Grafana


               APPLICATION LOGS
                     |
                 Alloy
                     |
                   Loki
                     |
                  Grafana
