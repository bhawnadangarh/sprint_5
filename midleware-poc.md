# POC – Redis Monitoring

<img width="300" height="300" alt="redis monitoring" src="https://github.com/user-attachments/assets/8b79c592-daaa-44cd-8150-1e51b1adff74" />

---

## Author Information

| Author | Created | Version | Last Updated By | Last Edited On | L0 Reviewer | L1 Reviewer | L2 Reviewer |
|---|---|---|---|---|---|---|---|
| Bhawna Dangarh | 2026-08-11 | 1.0 | Bhawna Dangarh | 2026-08-11 | Sharvari Khamkar / Tina Bhatnagar | Aman Raj | Abhishek Dubey |

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Objective](#2-objective)
3. [Step-by-Step Implementation](#3-step-by-step-implementation)  
   3.1 [Install and Start Redis](#31-install-and-start-redis)  
   3.2 [Install Redis Exporter](#32-install-redis-exporter)  
   3.3 [Install and Configure Prometheus](#33-install-and-configure-prometheus)  
   3.4 [Import Dashboard](#34-import-dashboard)  
   3.5 [Configure Alerts](#35-configure-alerts)  
4. [Commands Used](#4-commands-used)
5. [Troubleshooting](#5-troubleshooting)
6. [Contact Information](#6-contact-information)
7. [References](#7-references)

---

## 1. Introduction

This document details a Proof of Concept (POC) demonstrating the collection, visualization, and alerting setup for Redis database metrics (Memory Usage, Connected Clients, Evictions, and Expired Keys) using Prometheus Redis Exporter and Grafana.

Monitoring Redis is critical to prevent caching exhaustion and ensure backend response times remain optimal.

---

## 2. Objective

- Set up a running Redis instance.
- Deploy Prometheus Redis Exporter to translate Redis INFO metrics into Prometheus format.
- Configure Prometheus to pull metrics from Redis Exporter.
- Visualize caching performance using Grafana dashboard.
- Establish alerts for high memory utilization and connection spikes.

---

## 3. Step-by-Step Implementation

### 3.1 Install and Start Redis

Install Redis server on the Linux instance.

```bash
sudo apt-get update
sudo apt-get install -y redis-server
```


Verify service is up:

```bash
redis-cli ping
```
<img width="1919" height="403" alt="image" src="https://github.com/user-attachments/assets/4bc9ed7e-0788-4975-a67a-944e7c23d383" />
<img width="1642" height="306" alt="image" src="https://github.com/user-attachments/assets/edca53e1-c5e3-4dd8-b793-3195c6b4eb2b" />

---

### 3.2 Install Redis Exporter

Download and run Prometheus Redis Exporter.

```bash
wget https://github.com/oliver006/redis_exporter/releases/download/v1.55.0/redis_exporter-v1.55.0.linux-amd64.tar.gz
tar -xvf redis_exporter-v1.55.0.linux-amd64.tar.gz
cd redis_exporter-v1.55.0.linux-amd64
./redis_exporter -redis.addr=redis://localhost:6379 &
```

Verify metrics are exposed on port 9121:

```bash
curl http://localhost:9121/metrics
```
<img width="1919" height="958" alt="image" src="https://github.com/user-attachments/assets/50c1d113-f9fc-40ae-a0f9-2a4af71bf1b9" />

## Start Redis Exporter

```bash
./redis_exporter -redis.addr=redis://localhost:6379 &
```

---

### 3.3 Install and Configure Prometheus

Download and install Prometheus server on the Linux instance.

```bash
wget https://github.com/prometheus/prometheus/releases/download/v2.45.0/prometheus-2.45.0.linux-amd64.tar.gz
tar -xvf prometheus-2.45.0.linux-amd64.tar.gz
cd prometheus-2.45.0.linux-amd64
```
<img width="1919" height="630" alt="image" src="https://github.com/user-attachments/assets/3677f0cc-548b-493e-bb69-b2ed4a79980e" />

Configure `prometheus.yml` to scrape the Redis Exporter targets and route alerts to Alertmanager:

```yaml
global:
  scrape_interval: 15s

# Connect Prometheus to Alertmanager
alerting:
  alertmanagers:
    - static_configs:
        - targets:
            - 'localhost:9093' # Default Alertmanager port

# Load Redis alert rules
rule_files:
  - "redis.alerts.yml"

# Scrape targets
scrape_configs:
  - job_name: 'redis-exporter'
    static_configs:
      - targets: ['localhost:9121']
```

Start the Prometheus service:

```bash
./prometheus --config.file=prometheus.yml &
```
<img width="1919" height="745" alt="image" src="https://github.com/user-attachments/assets/af660a1f-6056-4cc0-84ac-0dc46f4f9670" />

---



### 3.4 Install Grafana

Install and start the Grafana visualization server.

```bash
sudo apt-get install -y apt-transport-https software-properties-common wget

wget -q -O - https://packages.grafana.com/gpg.key | gpg --dearmor | sudo tee /usr/share/keyrings/grafana.gpg > /dev/null

echo "deb [signed-by=/usr/share/keyrings/grafana.gpg] https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list

sudo apt-get update
sudo apt-get install -y grafana


sudo systemctl enable grafana-server
sudo systemctl start grafana-server
sudo systemctl status grafana-server
```

<img width="1919" height="694" alt="image" src="https://github.com/user-attachments/assets/46d4eede-00ed-4626-befc-7949f8e3e2d0" />
<img width="1918" height="599" alt="image" src="https://github.com/user-attachments/assets/cddb2986-1ce2-48e4-8572-e95f8e40cbcb" />


## Grafana Login

Open the following URL in your browser:

```text
http://<EC2-PUBLIC-IP>:3000
```

Example:

```text
http://54.160.238.4:3000
```

<img width="1919" height="966" alt="image" src="https://github.com/user-attachments/assets/7f38feae-7447-4b27-beb0-912c9d4de2f8" />


### 3.4 Import Dashboard

Log in to Grafana, add Prometheus as a Data Source, and import the Redis Dashboard (ID: `763`).

1. Add your Prometheus datasource url `http://localhost:9090`.
2. Go to **Dashboards -> Import**.
3. Enter Dashboard ID `763` and choose the Prometheus datasource.

<img width="1919" height="945" alt="image" src="https://github.com/user-attachments/assets/ab7dcc3a-e964-46bd-b93f-fe4384031cd1" />

---

### 3.5 Configure Alerts

1. Create a Prometheus alerting rules file named `redis.alerts.yml`:

```yaml
groups:
  - name: redis-alerts
    rules:
      # Alert when Redis memory usage exceeds 85% of maxmemory
      - alert: RedisOutOfMemory
        expr: redis_memory_used_bytes / redis_memory_max_bytes * 100 > 85
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Redis Out of Memory (instance {{ $labels.instance }})"
          description: "Redis memory usage is > 85%. Current utilization is {{ $value | printf "%.2f" }}%."

      # Alert when too many clients connect
      - alert: RedisTooManyConnections
        expr: redis_connected_clients > 5000
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "Redis Too Many Connections (instance {{ $labels.instance }})"
          description: "Active connections exceeded 5000. Current connections: {{ $value }}."
```

Verify alert rules syntax:

```bash
promtool check rules redis.alerts.yml
```

2. Download and install Alertmanager:

```bash
wget https://github.com/prometheus/alertmanager/releases/download/v0.27.0/alertmanager-0.27.0.linux-amd64.tar.gz
tar -xvf alertmanager-0.27.0.linux-amd64.tar.gz
cd alertmanager-0.27.0.linux-amd64
```

3. Create the Alertmanager configuration file `alertmanager.yml` to route alerts to email:

```yaml
global:
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alertmanager@mygurukulam.co'
  smtp_auth_username: 'alertmanager@mygurukulam.co'
  smtp_auth_password: 'your-secure-smtp-password-or-app-password'
  smtp_require_tls: true

route:
  group_by: ['alertname', 'instance']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h
  receiver: 'email-receiver'

receivers:
  - name: 'email-receiver'
    email_configs:
      - to: 'bhawna.dangarh.snaatak@mygurukulam.co'
        send_resolved: true
```

4. Start Alertmanager:

```bash
./alertmanager --config.file=alertmanager.yml &
```

---

## 4. Commands Used

| Command | Description |
| --- | --- |
| `sudo apt-get install redis-server` | Installs Redis package on the host |
| `redis-cli ping` | Verifies Redis engine connection |
| `./redis_exporter ... &` | Launches Redis exporter background daemon |
| `curl http://localhost:9121/metrics` | Queries exposed Redis exporter metrics |
| `./prometheus ... &` | Launches Prometheus server in background |
| `promtool check rules <file>` | Validates YAML syntax of Prometheus alerts |
| `./alertmanager ... &` | Launches Alertmanager background daemon |
| `curl -X POST .../-/reload` | Dynamically reloads Prometheus settings |

---

## 5. Troubleshooting

| Issue | Solution |
| --- | --- |
| Redis Exporter shows scraper errors | Ensure Redis connection string is correct and Redis is running |
| Grafana shows "No Data" for memory | Ensure `maxmemory` is configured in `/etc/redis/redis.conf` to avoid divide-by-zero |
| Exporter port conflict | Configure alternative port using `-web.listen-address` flag if port 9121 is busy |
| Authentication failure | Pass password using `-redis.password` flag if Redis requires auth |
| Alertmanager not receiving alerts | Ensure the `alertmanagers` target is configured under `alerting` in `prometheus.yml` and port 9093 is open |
| Email notifications failing | Verify SMTP host, credentials, TLS configuration, and check Alertmanager log output |

---

## 6. Contact Information

| Name | Email |
| --- | --- |
| Bhawna Dangarh | [bhawna.dangarh.snaatak@mygurukulam.co](mailto:bhawna.dangarh.snaatak@mygurukulam.co) |

---

## 7. References

| Description | Link |
| --- | --- |
| Redis Exporter GitHub Repository | [https://github.com/oliver006/redis_exporter](https://github.com/oliver006/redis_exporter) |
| Redis Configuration Guide | [https://redis.io/docs/management/config/](https://redis.io/docs/management/config/) |
| Grafana Redis Dashboard Page | [https://grafana.com/grafana/dashboards/763](https://grafana.com/grafana/dashboards/763) |

---
