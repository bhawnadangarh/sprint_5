# Rolling Deployment Strategy | Documentation

## Document Details

| Author | Created | Version | Last Updated By | Last Edited On | L0 Reviewer | L1 Reviewer | L2 Reviewer |
|--------|---------|---------|-----------------|----------------|-------------|-------------|-------------|
| Bhawna Dangarh | 2026-08-11 | 1.0 | Bhawna Dangarh | 2026-08-11 | Sharvari Khamkar / Tina Bhatnagar | Aman Raj | Abhishek Dubey |

---

# Table of Contents

1. [Introduction](#1-introduction)
2. [How It Works (Workflow & Steps)](#2-how-it-works-workflow-steps)
3. [Benefits](#3-benefits)
4. [Risks & Mitigations](#4-risks-mitigations)
5. [Suitable Use Cases](#5-suitable-use-cases)
6. [Conclusion](#6-conclusion)
7. [Contact Information](#7-contact-information)
8. [References](#8-references)

---

# 1. Introduction

A **Rolling Deployment** is a deployment strategy where an application's running instances are gradually replaced with a newer version. During a rolling update, the system scales up new instances running the updated version while simultaneously scaling down/terminating old instances running the previous version. 

This strategy prevents service downtime because there are always healthy instances available to process incoming user requests.

---

# 2. How It Works (Workflow & Steps)

The rolling deployment process operates incrementally based on a configurable **Batch Size** (or concurrency rate):

```mermaid
graph TD
    Start([Start Rolling Deployment]) --> GetBatch[1. Select Batch of Instances to Update]
    GetBatch --> ProvisionNew[2. Provision New Instances with New Version]
    ProvisionNew --> HealthCheck{3. Check Health of New Batch?}
    HealthCheck -- Fail --> Rollback[4. Rollback: Terminate New, Restore Old]
    Rollback --> NotifyFail[Notify Team: Deployment Failed]
    HealthCheck -- Pass --> RouteTraffic[5. Shift Traffic to New Instances]
    RouteTraffic --> TerminateOld[6. Terminate Same Number of Old Instances]
    TerminateOld --> CheckMore{7. Any Old Instances Remaining?}
    CheckMore -- Yes --> GetBatch
    CheckMore -- No --> Complete([Deployment Success])
```

### Detailed Steps:
1. **Define Batch Parameters**: The deployment orchestrator determines the update batch size (e.g., 25% of total capacity or 1 instance at a time).
2. **Provision New Version**: The orchestrator launches the new version of the application on new or designated target nodes.
3. **Perform Health Checks**: The system monitors the startup process and verifies health status (using health endpoints like `/health`).
4. **Shift Traffic & Terminate Old**: Once health check passes, traffic is routed to the new instances, and an equivalent batch of old instances is terminated.
5. **Repeat**: The process repeats for the next batch until 100% of the nodes run the new version.

---

# 3. Benefits

- **Zero Downtime**: Since instances are replaced incrementally, the application continues serving traffic throughout the deployment.
- **Low Resource Overhead**: Unlike Blue-Green deployment, which requires doubling the compute capacity temporarily, a rolling deployment only requires a small buffer of extra nodes (equivalent to the batch size).
- **Early Feedback**: Problems (like memory leaks or start-up failures) in the new release can be caught during the first batch update, minimizing impact.

---

# 4. Risks & Mitigations

| Risk | Description | Mitigation Strategy |
| :--- | :--- | :--- |
| **Version Skew** | During deployment, both old and new versions run concurrently, which can cause session inconsistencies or database schema mismatches. | Design applications to be backward and forward-compatible, use session stickiness, and apply database migrations using the expand-contract pattern. |
| **Slow Rollouts** | Updating nodes batch-by-batch can take a long time for large clusters. | Tune batch size percentage (e.g., 25% or 50% instead of 1 instance at a time) to balance speed and safety. |
| **Rollback Complexity** | If a failure happens midway, the cluster will be in a mixed state. | Automate rollbacks where the orchestrator stops the rollout and reverses the process to restore old instances. |

---

# 5. Suitable Use Cases

- **Web Microservices**: Stateless applications where multiple versions can coexist without side effects.
- **Resource-Constrained Environments**: Environments where provisioning duplicate staging infrastructure is cost-prohibitive.
- **Continuous Deployment (CD) Pipelines**: High-frequency updates where automated, minor bug fixes are rolled out incrementally.

---

# 6. Conclusion

Rolling deployments provide a highly efficient, zero-downtime release mechanism with minimal infrastructure overhead. By configuring proper batch sizes and rigorous health checks, teams can safely deploy updates with automated fallback logic.

---

# 7. Contact Information

| Name | Email | Role |
|------|-------|------|
| Bhawna Dangarh | bhawna.dangarh.snaatak@mygurukulam.co | DevOps Engineer |

---

# 8. References

| Source | Description |
|--------|-------------|
| Kubernetes Deployment | https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-update-deployment |
| AWS Autoscaling Rolling Updates | https://docs.aws.amazon.com/autoscaling/ec2/userguide/as-rolling-updates.html |
