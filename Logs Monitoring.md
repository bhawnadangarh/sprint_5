# POC – Application Logs Monitoring

<img width="400" height="400" alt="image" src="https://github.com/user-attachments/assets/2a70024a-507e-4065-9660-ef729b724294" />

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
   3.1 [Install Grafana Loki](#31-install-grafana-loki)  
   3.2 [Install and Configure Promtail](#32-install-and-configure-promtail)  
   3.3 [Configure Grafana Loki Datasource](#33-configure-grafana-loki-datasource)  
   3.4 [Visualize Log Metrics](#34-visualize-log-metrics)  
   3.5 [Configure Log-Based Alerts](#35-configure-log-based-alerts)  
4. [Commands Used](#4-commands-used)
5. [Troubleshooting](#5-troubleshooting)
6. [Contact Information](#6-contact-information)
7. [References](#7-references)

---

## 1. Introduction

This document details a Proof of Concept (POC) demonstrating the centralized collection, parsing, visualization, and alerting setup for application logs using Grafana Loki (log aggregation engine) and Promtail (log shipper agent).

Logs are parsed dynamically to generate metric counts (e.g. rate of `ERROR` logs) which can trigger operations team pages.

---

## 2. Objective

- Set up Grafana Loki log engine.
- Configure Promtail to tail application log files (e.g., Spring Boot, Python, or Go APIs) and ship them to Loki.
- Query logs in Grafana using LogQL.
- Create dashboard panel visualizations for log rate patterns and error trends.
- Setup LogQL alert rules to trigger when critical errors spike.

---

## 3. Step-by-Step Implementation

### 3.1 Install Grafana Loki

Download the Loki binary and config, and launch it.

```bash
sudo apt update
sudo apt install -y unzip
wget https://github.com/grafana/loki/releases/download/v2.9.4/loki-linux-amd64.zip
unzip loki-linux-amd64.zip
chmod +x loki-linux-amd64

wget https://raw.githubusercontent.com/grafana/loki/v2.9.4/cmd/loki/loki-local-config.yaml
./loki-linux-amd64 -config.file=loki-local-config.yaml &
```

<img width="1919" height="455" alt="image" src="https://github.com/user-attachments/assets/c276c9e3-7fe0-4e27-8250-8ebe2533d925" />
<img width="1918" height="465" alt="image" src="https://github.com/user-attachments/assets/8dc87761-db8d-48d1-b431-1df2221ff370" />

Verify Loki is listening on port 3100:

```bash
curl http://localhost:3100/ready
```
<img width="961" height="86" alt="image" src="https://github.com/user-attachments/assets/d2b3e315-0cfe-4854-9cda-d6d5780c5c5d" />


---

### 3.2 Install and Configure Promtail

Download the Promtail agent and create a configuration file (`promtail-local-config.yaml`).

```bash
wget https://github.com/grafana/loki/releases/download/v2.9.4/promtail-linux-amd64.zip
unzip promtail-linux-amd64.zip
chmod +x promtail-linux-amd64
```
<img width="1919" height="796" alt="image" src="https://github.com/user-attachments/assets/0a2c54d3-8da4-4b58-aaeb-57ac8468e97d" />

Create `promtail-local-config.yaml` to tail the target application log file:

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://localhost:3100/loki/api/v1/push

scrape_configs:
- job_name: application-logs
  static_configs:
  - targets:
      - localhost
    labels:
      job: otms-api-logs
      __path__: /var/log/otms/*.log
```

Start the Promtail daemon:

```bash
./promtail-linux-amd64 -config.file=promtail-local-config.yaml &
```

---

### 3.3 Configure Grafana Loki Datasource

1. Open your Grafana console (port 3000).
2. Go to **Connections -> Data Sources** -> **Add data source** -> Select **Loki**.
3. Set the Connection URL to `http://localhost:3100`.
4. Click **Save & test** to verify successful integration.

---

### 3.4 Visualize Log Metrics

In Grafana, navigate to the **Explore** tab or create a new Dashboard panel. You can convert log lines into metric counts using LogQL queries:

- **Error Log Rate (Metrics from logs)**:
  ```logql
  sum(count_over_time({job="otms-api-logs"} |~ "ERROR" [5m]))
  ```
- **Total Application Log Throughput**:
  ```logql
  sum(rate({job="otms-api-logs"}[5m]))
  ```

---

### 3.5 Configure Log-Based Alerts

1. Add alerting rules based on log patterns inside your Loki configuration file under the `ruler` section:

```yaml
ruler:
  storage:
    type: local
    local:
      directory: /tmp/loki/rules
  rule_path: /tmp/loki/scratch
  alertmanager_url: http://localhost:9093
  ring:
    kvstore:
      store: inmemory
```

Create Loki alerting rules in a separate file (e.g. `/tmp/loki/rules/fake/logs-alerts.yml`):

```yaml
groups:
  - name: application-logs-alerts
    rules:
      - alert: CriticalLogErrorsSpike
        expr: sum(count_over_time({job="otms-api-logs"} |~ "CRITICAL|FATAL|ERROR" [5m])) > 10
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Spike in Critical Errors detected in logs"
          description: "There are more than 10 critical errors/exceptions in the logs during the last 5 minutes."
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
| `wget https://github.com/grafana/loki/releases/download/...` | Downloads Loki/Promtail zip package |
| `unzip <file>.zip` | Extracts zipped Promtail and Loki binaries |
| `./loki-linux-amd64 -config.file=... &` | Starts the Loki aggregation engine in background |
| `./promtail-linux-amd64 -config.file=... &` | Starts Promtail agent to begin forwarding logs |
| `curl http://localhost:3100/ready` | Verifies Loki availability status |
| `wget https://github.com/prometheus/alertmanager/...` | Downloads Alertmanager package |
| `tar -xvf <file>.tar.gz` | Extracts Alertmanager package |
| `./alertmanager --config.file=... &` | Launches Alertmanager background daemon |

---

## 5. Troubleshooting

| Issue | Solution |
| --- | --- |
| Promtail fails to tail file | Ensure permissions allow reading `/var/log/otms/*.log` by the promtail process |
| Loki target "down" in Promtail | Verify network configuration and check that Loki is running on port 3100 |
| LogQL query returns "No Data" | Ensure the application has written logs and Promtail positions directory has write permission |
| Promtail stuck reading old files | Clear the positions tracking database file (`/tmp/positions.yaml`) and restart |
| Alertmanager not receiving alerts | Ensure the `alertmanager_url` target is configured under `ruler` in Loki local config file and port 9093 is open |
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
| Grafana Loki Getting Started | [https://grafana.com/docs/loki/latest/get-started/](https://grafana.com/docs/loki/latest/get-started/) |
| Promtail Configuration Guide | [https://grafana.com/docs/loki/latest/send-data/promtail/configuration/](https://grafana.com/docs/loki/latest/send-data/promtail/configuration/) |
| LogQL Query Language Reference | [https://grafana.com/docs/loki/latest/query/](https://grafana.com/docs/loki/latest/query/) |

---
