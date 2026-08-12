# App Monitoring Alerting Rules & Process

<img width="400" height="300" alt="image" src="https://github.com/user-attachments/assets/41fba51d-fe7e-4506-9ae3-0c3ac26c8dab" />

## Document Details

| Author         | Created    | Version | Last Updated By | Last Edited On | L0 Reviewer                       | L1 Reviewer | L2 Reviewer    |
| -------------- | ---------- | ------- | --------------- | -------------- | --------------------------------- | ----------- | -------------- |
| Bhawna Dangarh | 2026-08-11 | 1.0     | Bhawna Dangarh  | 2026-08-11     | Sharvari Khamkar / Tina Bhatnagar | Aman Raj    | Abhishek Dubey |

---

# Table of Contents

1. [Introduction](#1-introduction)
2. [Alerting Rules (Prometheus Syntax)](#2-alerting-rules-prometheus-syntax)
3. [Severity Levels](#3-severity-levels)
4. [Notification Channels & Routing](#4-notification-channels--routing)
5. [Incident Escalation Process](#5-incident-escalation-process)
6. [Conclusion](#6-conclusion)
7. [FAQs](#7-faqs)
8. [Contact Information](#8-contact-information)
9. [References](#9-references)

---

# 1. Introduction

This document defines the monitoring and alerting process for **OTMS microservices**. It covers application health, HTTP error rates, response latency, alert severity, notification routing, and incident escalation.

**Prometheus** detects application issues, **Alertmanager** routes alerts based on severity, and the **L0 → L1 → L2** escalation process ensures timely incident response and resolution.

---

# 2. Alerting Rules (Prometheus Syntax)

These rules monitor application status, error rates, and response latencies.

### A. Application Down Alert

Triggers when an application service instance goes offline.

```yaml
groups:
  - name: application-health
    rules:
      - alert: AppInstanceDown
        expr: up{job="otms-services"} == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Application instance {{ $labels.instance }} down"
          description: "Service {{ $labels.job }} is offline for more than 2 minutes."
```

### B. High HTTP 5xx Error Rate

Triggers when HTTP 5xx error responses exceed 5% of total requests over a 5-minute window.

```yaml
      - alert: HighHttp5xxErrorRate
        expr: sum(rate(http_requests_total{status=~"5.."}[5m])) / sum(rate(http_requests_total[5m])) * 100 > 5
        for: 3m
        labels:
          severity: warning
        annotations:
          summary: "High HTTP 5xx Error Rate detected"
          description: "HTTP 5xx error rate is {{ $value | printf \"%.2f\" }}% on application service."
```

### C. Slow HTTP Response Latency

Triggers when the 95th percentile response latency exceeds 2 seconds.

```yaml
      - alert: SlowHttpResponseLatency
        expr: histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le)) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Slow Response Latency (p95 > 2s)"
          description: "p95 HTTP response duration is {{ $value }}s over the last 5 minutes."
```

---

# 3. Severity Levels

We classify alerts into three severity tiers:

| Severity     | Definition                                                    | Action Timeline                           | Example Incident                               |
| :----------- | :------------------------------------------------------------ | :---------------------------------------- | :--------------------------------------------- |
| **Critical** | Production system is down or severely degraded.               | Immediate action. 15-minute response SLA. | `AppInstanceDown`, database connection failure |
| **Warning**  | Application is functional but experiencing errors or latency. | Investigation within 4 hours.             | `HighHttp5xxErrorRate`, high CPU               |
| **Info**     | Non-critical configuration or operational condition.          | Routine review.                           | SSL certificate expiring in 30 days            |

---

# 4. Notification Channels & Routing

<img width="1536" height="800" alt="image" src="https://github.com/user-attachments/assets/36eab7a5-3a5e-4745-9f42-cd01b1b1300e" />

### Alertmanager Routing Configuration Snippet

```yaml
route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h
  receiver: 'slack-warnings'
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty-critical'
    - match:
        severity: warning
      receiver: 'slack-warnings'

receivers:
  - name: 'slack-warnings'
    slack_configs:
      - channel: '#prod-alerts'
        api_url: 'https://hooks.slack.com/services/T00/B00/X00'

  - name: 'pagerduty-critical'
    pagerduty_configs:
      - service_key: 'pd-service-routing-key-example'
```

---

# 5. Incident Escalation Process

<img width="1672" height="800" alt="image" src="https://github.com/user-attachments/assets/3d182a49-09f7-408e-8b9b-c62c282a590f" />

### Escalation Path Details

1. **L0 Support (Primary On-Call):** Has **15 minutes** to acknowledge the critical alert and start investigation.
2. **L1 Support (Lead Engineers):** If L0 does not respond, the incident escalates to L1, who has another **15 minutes** to work toward resolution.
3. **L2 Management (Ops Managers):** If the incident remains unresolved, it escalates to L2 for management intervention.
4. **Still Unresolved:** Escalation and management actions continue until the incident is resolved and formally closed.

---

# 6. Conclusion

The monitoring and alerting process provides a structured approach to detect application issues, notify the right teams, and escalate unresolved incidents. Prometheus identifies application conditions, Alertmanager routes alerts based on severity, and the L0 → L1 → L2 process ensures timely incident response and resolution.

---

# 7. FAQs

| Question                                            | Answer                                                                                                 |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------------------ |
| **What is the purpose of this monitoring process?** | To detect application issues, notify the appropriate team, and ensure critical incidents are resolved. |
| **What does Prometheus do?**                        | Prometheus collects application metrics and evaluates predefined alerting rules.                       |
| **What does Alertmanager do?**                      | Alertmanager receives alerts from Prometheus and routes them based on severity.                        |
| **What happens if L0 does not respond?**            | The incident is automatically escalated to L1 after 15 minutes.                                        |
| **What happens if L1 cannot resolve the incident?** | The incident is escalated to L2 for further management intervention and continues until resolved.      |

---

# 8. Contact Information

| Name           | Email                                                                                 |
| -------------- | ------------------------------------------------------------------------------------- | 
| Bhawna Dangarh | [bhawna.dangarh.snaatak@mygurukulam.co](mailto:bhawna.dangarh.snaatak@mygurukulam.co) | 

---

# 9. References

| Source                     | Description                                         |
| -------------------------- | --------------------------------------------------- |
| Prometheus Alerting Rules  | Prometheus alerting rule configuration and syntax   |
| Alertmanager Configuration | Alertmanager routing and notification configuration |
