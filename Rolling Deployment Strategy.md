# Deployment Strategies

<img width="400" height="300" alt="image" src="https://github.com/user-attachments/assets/46d4f85f-d826-4167-9b4e-cbfcb4d5e438" />

---

## Author Information

| Author | Created | Version | Last Updated By | Last Edited On | L0 Reviewer | L1 Reviewer | L2 Reviewer |
|---|---|---|---|---|---|---|---|
| Bhawna Dangarh | 2026-08-11 | 1.3 | Bhawna Dangarh | 2026-08-11 | Sharvari Khamkar / Tina Bhatnagar | Aman Raj | Abhishek Dubey |

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Objective](#2-objective)
3. [Deployment Strategies Explained](#3-deployment-strategies-explained)  
   3.1 [Recreate Deployment](#31-recreate-deployment)  
   3.2 [Blue-Green Deployment](#32-blue-green-deployment)  
   3.3 [Canary Deployment](#33-canary-deployment)  
   3.4 [Shadow (A/B) Deployment](#34-shadow-ab-deployment)  
4. [Rolling Deployment – Detailed Strategy](#4-rolling-deployment-detailed-strategy)  
   4.1 [Rolling Deployment Flowchart](#41-rolling-deployment-flowchart)  
   4.2 [How It Works (Workflow & Steps)](#42-how-it-works-workflow--steps)  
   4.3 [Benefits](#43-benefits)  
   4.4 [Risks & Mitigations](#44-risks--mitigations)  
   4.5 [Suitable Use Cases](#45-suitable-use-cases)  
5. [Frequently Asked Questions (FAQs)](#5-frequently-asked-questions-faqs)
6. [Contact Information](#6-contact-information)
7. [References](#7-references)


---

## 1. Introduction

Deployment strategies define how a new application version is released to a production environment. The choice of strategy depends on factors such as application architecture, resource availability, budget, downtime requirements, and deployment risk. This document explains the commonly used deployment strategies and provides a detailed overview of Rolling Deployment, including its workflow, benefits, risks, and suitable use cases.

---

## 2. Objective

The objective of this document is to explain the commonly used deployment strategies, including Recreate, Blue-Green, Canary, and Shadow/A-B. It also provides a detailed overview of the Rolling Deployment strategy, covering its workflow, benefits, risks, and suitable use cases. Additionally, it explains how Rolling Deployment helps achieve continuous application releases with minimal downtime.

---

## 3. Deployment Strategies Explained

### 3.1 Recreate Deployment

In a **Recreate** deployment, all running instances of the current version are stopped before the new version is started. This approach is simple to configure and avoids running different versions at the same time. However, it causes downtime while the new containers are starting.

### 3.2 Blue-Green Deployment

In a **Blue-Green** deployment, two identical production-ready environments are maintained. The new release is deployed to the inactive environment (Green) and tested. Once it is verified, traffic is switched to the Green environment. This provides zero downtime and quick rollback, but it requires additional computing resources.

### 3.3 Canary Deployment

In a **Canary** deployment, the new version is first deployed to a small part of the production environment (for example, 5-10% of nodes). A small percentage of live users are routed to the new version to monitor its performance and behavior. If the release is stable, it is gradually deployed to the remaining environment. This helps reduce the impact of unexpected issues.

### 3.4 Shadow (A/B) Deployment

In a **Shadow (A/B)** deployment, the new version runs alongside the active version. Production traffic is copied and sent to both versions, but only the response from the active version is returned to users. This allows the new version to be tested with real production traffic without affecting users. However, it can be complex to configure.

---

## 4. Rolling Deployment Detailed Strategy

A **Rolling Deployment** replaces application instances gradually. A batch of new instances is started with the new version while old instances are removed step by step. This helps prevent service-wide downtime.

### 4.1 Rolling Deployment Flowchart

<img width="1309" height="800" alt="image" src="https://github.com/user-attachments/assets/1c002b3e-17ec-4de0-88b7-a36604000c40" />

### 4.2 How It Works (Workflow & Steps)

1. **Batch Sizing**: Configure parameters such as `max_surge` (extra instances launched) and `max_unavailable` (number of old nodes that can be removed at one time). For example, a 25% batch size on a 4-node cluster updates 1 node at a time.

2. **Launch New Instances**: The orchestrator starts new instances with the updated package or container image.

3. **Health Validation**: The system checks the health of the new instances. For example, the `/health` endpoint on port `8080/8081` should return `200 OK` before the instances are marked as active.

4. **Traffic Redirect**: The load balancer sends user traffic to the healthy and validated new instances.

5. **Decommission Old Nodes**: The same number of old-version instances are removed after the new instances become healthy.

6. **Iterate**: The process continues until 100% of the active instances are running the new release version.

### 4.3 Benefits

| Benefit | Description |
| :--- | :--- |
| **Zero Downtime** | Active instances remain available to serve user requests during the deployment. |
| **Low Overhead** | Only a small number of extra resources are required during the update, making it more cost-effective than Blue-Green. |
| **Fail-Fast Safety** | Problems with the new version can be detected during the first batch, reducing the impact of failures. |
| **Easy Monitoring** | Each new batch can be checked using health checks and application metrics before continuing. |
| **Gradual Release** | The new version is introduced step by step, making the deployment easier to control. |

### 4.4 Risks & Mitigations

- **Version Skew:** Old and new versions run at the same time, which can cause session issues or database schema conflicts.  
  **Mitigation:** Ensure APIs are backward and forward-compatible. Use sticky sessions on load balancers and follow the expand-contract pattern for database migrations.

- **Slow Rollout:** Large host groups can take more time to upgrade.  
  **Mitigation:** Increase the batch size when it is safe to do so, for example, updating 50% of instances instead of 10%.

- **State/Session Loss:** User sessions on terminated hosts may be lost.  
  **Mitigation:** Store application state and sessions in an external system such as Redis.

### 4.5 Suitable Use Cases

| Recommended Use Case | Scenario & Criteria | Examples |
| :--- | :--- | :--- |
| **Stateless Microservices** | Suitable for web APIs and microservices where multiple versions can run at the same time without session or database compatibility issues. | React Frontends, Salary/Attendance APIs |
| **High-Availability APIs** | Suitable for critical backend APIs where zero downtime is required. | Production APIs |
| **Resource-Constrained Environments** | Suitable for small environments or environments with limited budgets where running duplicate infrastructure like Blue-Green is expensive. | Small AWS environments |
| **Small, Frequent Updates** | Suitable for CI/CD pipelines that frequently deploy small fixes or feature updates. | CI/CD applications |

---

## 5. Frequently Asked Questions (FAQs)

| Question | Answer / Explanation |
| :--- | :--- |
| **Q1: Which deployment strategy is the most cost-effective?** | **Recreate** requires no extra resources. **Rolling** is also cost-effective because it needs only a small temporary resource buffer, while **Blue-Green** requires additional infrastructure. |
| **Q2: How does Rolling Deployment prevent downtime?** | It updates instances in small batches while healthy instances continue serving traffic. |
| **Q3: What is the main risk of Rolling Updates?** | **Version skew**, where old and new versions run at the same time and may cause compatibility issues. |
| **Q4: When should we choose Canary instead of Rolling?** | Choose **Canary** for high-risk or major updates that need to be tested with a small percentage of live users first. |
| **Q5: Can we rollback a Rolling Deployment instantly?** | No. The previous version usually needs to be deployed again across the nodes batch by batch. |

---

## 6. Contact Information

| Name | Email |
|---|---|
| Bhawna Dangarh | [bhawna.dangarh.snaatak@mygurukulam.co](mailto:bhawna.dangarh.snaatak@mygurukulam.co) |

---

## 7. References

| Source | Description | Link |
|---|---|---|
| AWS ASG Instance Refresh | AWS guide on performing rolling instance updates | [AWS Documentation](https://docs.aws.amazon.com/autoscaling/ec2/userguide/asg-instance-refresh.html) |
| Cloud-Native Deployment Strategies | CNCF guide on modern cloud deployment patterns | [CNCF Blog](https://www.cncf.io/blog/2022/10/24/kubernetes-deployment-strategies-rolling-blue-green-canary/) |
