# POC – Infrastructure Monitoring

<img width="300" height="300" alt="infrastructure monitoring" src="https://github.com/user-attachments/assets/8b79c592-daaa-44cd-8150-1e51b1adff74" />

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

Verify that metrics are being exposed locally on port 9100:

```bash
curl http://localhost:9100/metrics
```

---

### 3.2 Configure Prometheus Target

Add the Node Exporter target to your Prometheus server configuration file (`prometheus.yml`).

```yaml
scrape_configs:
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['localhost:9100']
```

Reload Prometheus configuration:

```bash
curl -X POST http://localhost:9090/-/reload
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

sudo systemctl enable grafana-server
sudo systemctl start grafana-server
```

---

### 3.4 Import Dashboard

Log in to Grafana (port 3000), add Prometheus as a Data Source, and import the standard Node Exporter Dashboard (ID: `1860`).

1. Navigate to **Connections -> Data Sources** -> Click **Add data source** -> Select **Prometheus**.
2. Set URL to `http://localhost:9090` and click **Save & test**.
3. Go to **Dashboards -> New -> Import**.
4. Enter Dashboard ID `1860` and select the Prometheus datasource.

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

Verify alert rule files syntax using `promtool`:

```bash
promtool check rules alert.rules.yml
```

---

## 4. Commands Used

| Command | Description |
| --- | --- |
| `tar -xvf` | Extracts the downloaded Node Exporter archive |
| `./node_exporter &` | Launches Node Exporter in the background |
| `curl http://localhost:9100/metrics` | Queries exposed node exporter metrics |
| `sudo systemctl start grafana-server` | Starts the Grafana service daemon |
| `promtool check rules <file>` | Validates Prometheus alert rules syntax |
| `curl -X POST .../-/reload` | Dynamically reloads Prometheus settings |

---

## 5. Troubleshooting

| Issue | Solution |
| --- | --- |
| Node Exporter service fails to start | Check port 9100 usage using `ss -lntp` or `netstat -plnt` |
| Prometheus shows target in "DOWN" state | Verify network/firewall access on port 9100 of the target host |
| Grafana shows "No Data" | Ensure the dashboard query is referencing the correct Prometheus datasource |
| Prometheus alert rules validation error | Ensure indentation matches standard YAML syntax and run `promtool check` |

---

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
