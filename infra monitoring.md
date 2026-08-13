# POC – Infrastructure Monitoring

<img width="400" height="300" alt="image" src="https://github.com/user-attachments/assets/e0a0f22e-98a0-4127-8dc6-53c47220ad32" />

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
   3.1 [Install Node Exporter](#31-install-node-exporter)  
   3.2 [Configure Prometheus Target](#32-configure-prometheus-target)  
   3.3 [Install Grafana](#33-install-grafana)  
   3.4 [Import Dashboard](#34-import-dashboard)  
   3.5 [Configure Alerts](#35-configure-alerts)  
4. [Commands Used](#4-commands-used)
5. [Troubleshooting](#5-troubleshooting)
6. [Contact Information](#6-contact-information)
7. [References](#7-references)

---

## 1. Introduction

This document details a Proof of Concept (POC) demonstrating the automated collection, visualization, and alerting setup for key infrastructure metrics (CPU, Memory, Disk, and Network) using Prometheus Node Exporter, Prometheus Server, and Grafana.

Monitoring infrastructure health ensures that resource exhaustion is detected before causing application downtime.

---

## 2. Objective

- Collect system-level metrics from Linux hosts using Node Exporter.
- Configure Prometheus to pull metrics at regular scrape intervals.
- Visualize system performance using Grafana dashboards.
- Set up alerting rules for high CPU, low memory, and disk space saturation.
- Test endpoint metrics scrape target success.

---

## 3. Step-by-Step Implementation

### 3.1 Install Node Exporter

Download and install Prometheus Node Exporter on the target Linux instance.

```bash
wget https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
tar -xvf node_exporter-1.7.0.linux-amd64.tar.gz
cd node_exporter-1.7.0.linux-amd64
./node_exporter &
```
<img width="1919" height="849" alt="image" src="https://github.com/user-attachments/assets/27a1f086-48f2-4cc0-832d-0ff894271caa" />

Verify that metrics are being exposed locally on port 9100:

```bash
curl http://localhost:9100/metrics
```
<img width="1662" height="721" alt="image" src="https://github.com/user-attachments/assets/41c40ec3-db6b-4f9e-9cf2-66587a0327da" />



### 3.2 Install Prometheus

Install and start Prometheus on the monitoring server.

```bash
sudo apt update
sudo apt install -y prometheus
````


Verify the Prometheus service:

```bash
sudo systemctl status prometheus
```

Verify that Prometheus is running on port `9090`:

```bash
curl http://localhost:9090/-/ready
```
<img width="1919" height="467" alt="image" src="https://github.com/user-attachments/assets/6a83ac4e-de13-4c6b-98f0-2cc6ff9338e7" />


<img width="1917" height="736" alt="image" src="https://github.com/user-attachments/assets/7998aa3b-797d-4854-9e31-f09b8c86bcb6" />

---

### 3.2 Configure Prometheus Target

Add the Node Exporter target to your Prometheus server configuration file (`prometheus.yml`).

```yaml
scrape_configs:
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['localhost:9100']
```

---

### 3.3 Install Grafana

Install and start the Grafana visualization server.

```bash
sudo apt-get install -y apt-transport-https software-properties-common wget
wget -q -O - https://packages.grafana.com/gpg.key | gpg --dearmor | sudo tee /usr/share/keyrings/grafana.gpg > /dev/null

echo "deb [signed-by=/usr/share/keyrings/grafana.gpg] https://packages.grafana.com/oss/deb stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list

sudo apt-get update
sudo apt-get install -y grafana


```
<img width="1919" height="490" alt="image" src="https://github.com/user-attachments/assets/848f8300-ab62-43b1-9263-d0e41942e6c7" />
<img width="1919" height="572" alt="image" src="https://github.com/user-attachments/assets/b72cd847-95e1-4942-8579-21019645adef" />

---

### 3.4 Import Dashboard

Log in to Grafana (port 3000), add Prometheus as a Data Source, and import the standard Node Exporter Dashboard (ID: `1860`).

1. Navigate to **Connections -> Data Sources** -> Click **Add data source** -> Select **Prometheus**.
2. Set URL to `http://localhost:9090` and click **Save & test**.
3. Go to **Dashboards -> New -> Import**.
4. Enter Dashboard ID `1860` and select the Prometheus datasource.

<img width="1912" height="965" alt="image" src="https://github.com/user-attachments/assets/a3c1269c-c6ef-4c27-8107-d85f0bf825bf" />
<img width="1918" height="965" alt="image" src="https://github.com/user-attachments/assets/bcafa37b-1b22-48a7-8d00-8be44125d957" />

<img width="1919" height="952" alt="image" src="https://github.com/user-attachments/assets/594ceffc-3f3f-4f9f-9d09-77e2570eb750" />
<img width="1918" height="967" alt="image" src="https://github.com/user-attachments/assets/be837b54-42de-4c96-a0cb-7fd645153e1c" />

---

### 3.5 Configure Alerts

Add infrastructure alert rules to your Prometheus configuration (e.g. `alert.rules.yml`).

```yaml
groups:
  - name: infra-alerts
    rules:
      # Alert for high CPU usage
      - alert: HostHighCpuLoad
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Host High CPU Load (instance {{ $labels.instance }})"
          description: "CPU load is > 85% for 5 minutes. Current load is {{ $value | printf \"%.2f\" }}%."

      # Alert for low memory space
      - alert: HostOutOfMemory
        expr: node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100 < 10
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Host Out Of Memory (instance {{ $labels.instance }})"
          description: "Available memory is < 10%. Current availability is {{ $value | printf \"%.2f\" }}%."
```

### 3.6 Install and Configure Alertmanager

Alertmanager receives alerts from Prometheus and sends notifications such as email alerts.

Install Alertmanager on the same monitoring server where Prometheus is running.

#### Install Alertmanager

```bash
sudo apt update
sudo apt install -y prometheus-alertmanager
````
<img width="1919" height="963" alt="image" src="https://github.com/user-attachments/assets/5fa5ae93-df31-4b09-9dda-a2e0b22966a8" />

#### Verify Alertmanager

```bash
sudo systemctl status prometheus-alertmanager
```

<img width="1887" height="542" alt="image" src="https://github.com/user-attachments/assets/f1defbd5-760d-4c60-b367-019c08a29486" />


#### Alertmanager Configuration

The Alertmanager configuration file is:

```bash
sudo vim /etc/prometheus/alertmanager.yml

```

```
global:
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'bhavna123porwal@gmail.com'
  smtp_auth_username: 'bhavna123porwal@gmail.com'
  smtp_auth_password: 'NEW_APP_PASSWORD'

route:
  receiver: 'email-notification'

receivers:
  - name: 'email-notification'
    email_configs:
      - to: 'bhavna123porwal@gmail.com'

```

#### Connect Prometheus to Alertmanager

Edit
```
sudo vim /etc/prometheus/prometheus.yml

```
```
rule_files:
  - "/etc/prometheus/alert.rules.yml"

```

```
 sudo vim /etc/default/prometheus-alertmanager
```
```
ARGS="--config.file=/etc/prometheus/alertmanager.yml"

```

## 6. Contact Information

| Name | Email |
| --- | --- |
| Bhawna Dangarh | [bhawna.dangarh.snaatak@mygurukulam.co](mailto:bhawna.dangarh.snaatak@mygurukulam.co) |

---

## 7. References

| Description | Link |
| --- | --- |
| Node Exporter GitHub Repository | [https://github.com/prometheus/node_exporter](https://github.com/prometheus/node_exporter) |
| Prometheus Scrape Configuration | [https://prometheus.io/docs/prometheus/latest/configuration/configuration/](https://prometheus.io/docs/prometheus/latest/configuration/configuration/) |
| Grafana Dashboard Library | [https://grafana.com/grafana/dashboards/](https://grafana.com/grafana/dashboards/) |
| Prometheus Rule Checking Guide | [https://prometheus.io/docs/prometheus/latest/command-line/promtool/](https://prometheus.io/docs/prometheus/latest/command-line/promtool/) |

---
