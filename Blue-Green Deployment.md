# Immutable Infrastructure Rollout – Blue-Green Deployment

<img width="400" height="300" alt="image" src="https://github.com/user-attachments/assets/5c6b1ab1-7359-42b4-8b77-f54b51ac00d4" />

## Author Information

| Author         | Created    | Version | Last Updated By | Last Edited On | L0 Reviewer                       | L1 Reviewer | L2 Reviewer    |
| -------------- | ---------- | ------- | --------------- | -------------- | --------------------------------- | ----------- | -------------- |
| Bhawna Dangarh | 2026-08-09 | 1.1     | Bhawna Dangarh  | 2026-08-11     | Sharvari Khamkar / Tina Bhatnagar | Aman Raj    | Abhishek Dubey |

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [Objective](#2-objective)
3. [Blue-Green Infrastructure Flowchart](#3-blue-green-infrastructure-flowchart)
4. [Step-by-Step Implementation](#4-step-by-step-implementation)  
   4.1 [Terraform Configuration Setup](#41-terraform-configuration-setup)  
   4.2 [Provision Active (Blue) Environment](#42-provision-active-blue-environment)  
   4.3 [Deploy Update to Inactive (Green) Environment](#43-deploy-update-to-inactive-green-environment)  
   4.4 [Verify Health Checks on Inactive Target Group](#44-verify-health-checks-on-inactive-target-group)  
   4.5 [Traffic Shift Listener Switch](#45-traffic-shift-listener-switch)  
   4.6 [Scale Down and Terminate Old Environment](#46-scale-down-and-terminate-old-environment)  
5. [Commands Used](#5-commands-used)
6. [Troubleshooting](#6-troubleshooting)
7. [FAQs](#7-faqs)
8. [Contact Information](#8-contact-information)
9. [References](#9-references)
    
---

## 1. Introduction

Continuous Delivery (CD) helps us deploy application releases automatically, safely, and reliably. Instead of changing running EC2 instances directly (Mutable Infrastructure), which can cause configuration changes and downtime, we use **Blue-Green Deployment** with **Immutable Infrastructure**.

In this setup, we keep two similar production environments:

* **Blue**: The current active production environment.
* **Green**: The inactive environment where the new version is deployed and tested before sending traffic to it.

If there is any issue with the new version, we can quickly roll back by pointing the load balancer back to the healthy Blue environment.

---

## 2. Objective

* Automate application deployments without downtime.
* Use Terraform to manage Target Groups, Auto Scaling Groups (ASGs), and Application Load Balancer (ALB) routing rules.
* Demonstrate the complete Blue-Green deployment using an automated shell script.
* Use health checks to make sure unhealthy application versions do not receive user traffic.
* Reduce resource usage by scaling the inactive environment to zero after a successful deployment.

---

## 3. Blue-Green Infrastructure Flowchart

The flowchart below shows the deployment steps and rollback points used in this POC:

<img width="1536" height="800" alt="image" src="https://github.com/user-attachments/assets/7889c9ca-bad6-437a-9cb1-b913d11755c1" />

---


## 5. Step-by-Step Manual Rollout via AWS Console

The following details the sequential lifecycle of resources when executing a Blue-Green deployment manually using the AWS Management Console:

### 5.1 Stage 1: Provision Initial Active Environment (Blue Only)
At the start, only the Blue environment resources are provisioned. The Application Load Balancer routes 100% of production traffic to this environment.

1. **Create Blue Target Group**:
   - Navigate to the **EC2 Console** -> **Target Groups** -> **Create Target Group**.
   - Target type: `Instances`. Name: `tg-otms-attendance-blue`. Port: `8081`. Protocol: `HTTP`.
   - Configure Health Checks: Path `/api/v1/attendance/health`, Protocol `HTTP`, Port `8081`.
2. **Create Launch Template**:
   - Go to **EC2** -> **Launch Templates** -> **Create Launch Template**.
   - Specify AMI ID (containing version `v1.0.0`), instance type (`t2.micro`), and Key Pair.
   - Configure Security Group (`dev-otms-attendance-sg`) to allow incoming traffic on port 8081.
3. **Create Blue Auto Scaling Group (ASG)**:
   - Go to **EC2** -> **Auto Scaling Groups** -> **Create Auto Scaling Group**.
   - Link the Blue Launch Template. Choose VPC and private backend subnets.
   - Under **Load Balancing**, select "Attach to an existing load balancer" and choose Target Group `tg-otms-attendance-blue`.
   - Set size: Desired = `2`, Min = `2`, Max = `5`.
4. **Configure ALB Listener Rule**:
   - Navigate to **EC2** -> **Load Balancers** -> Select your ALB (`dev-otms-alb`).
   - Go to the **Listeners and Rules** tab -> Select HTTP:80 Listener -> **Manage Rules** -> **Add Rule**.
   - Set condition: Path is `/api/v1/attendance*`.
   - Set action: Forward to Target Group `tg-otms-attendance-blue`. Priority: `20`.

Live users are now hitting the Blue target instances running version `v1.0.0`.

---

### 5.2 Stage 2: Introduce and Provision the Green Environment
When version `v2.0.0` is ready for release, we deploy it to the Green environment.

> [!IMPORTANT]
> At this stage, **do NOT edit the ALB Listener Rule**. The rule must remain pointing to `tg-otms-attendance-blue` so that production traffic is unaffected.

1. **Create Green Target Group**:
   - Go to **Target Groups** -> **Create Target Group**.
   - Target type: `Instances`. Name: `tg-otms-attendance-green`. Port: `8081`. Protocol: `HTTP`.
   - Set the same health check path: `/api/v1/attendance/health` on Port `8081`.
2. **Update Launch Template (Create New Version)**:
   - Go to **Launch Templates** -> Select your template -> **Modify Template (Create New Version)**.
   - Update the AMI ID to refer to the new image containing version `v2.0.0`.
3. **Create Green Auto Scaling Group (ASG)**:
   - Go to **Auto Scaling Groups** -> **Create Auto Scaling Group**.
   - Link the Launch Template and specify the new template version containing `v2.0.0`.
   - Under **Load Balancing**, choose Target Group `tg-otms-attendance-green`.
   - Set size: Desired = `2`, Min = `2`, Max = `5`.

Green instances are launched and automatically register themselves with `tg-otms-attendance-green`.

---

### 5.3 Stage 3: Health Status Check Verification
Before shifting production traffic, we verify that the Green instances are healthy:

1. Navigate to **EC2 Console** -> **Target Groups** -> Select `tg-otms-attendance-green`.
2. Under the **Targets** tab, view the status of the registered targets.
3. Wait until the health status changes from `initial` to **`healthy`**.

* **If healthy**: Proceed to Stage 4.
* **If unhealthy**: Stop the deployment. Rollback immediately by deleting/scaling down the Green ASG `asg-otms-attendance-green` size to `0` and investigate logs.

---

### 5.4 Stage 4: Shift Traffic to Green (Update ALB Listener Rule)
Once the Green targets are confirmed healthy, we redirect client traffic:

1. Navigate to **EC2** -> **Load Balancers** -> Select your ALB.
2. Go to **Listeners and Rules** -> Select the HTTP:80 listener -> View/Edit rules.
3. Select the rule with path `/api/v1/attendance*` and click **Edit**.
4. In the **Actions** section, change the target group from `tg-otms-attendance-blue` to **`tg-otms-attendance-green`**.
5. Save the rule.

The ALB immediately switches all routing to the Green Target Group, redirecting 100% of user traffic to version `v2.0.0`.

---

### 5.5 Stage 5: Scale Down Blue Environment
After monitoring the Green environment stability under production load for a designated window, scale down the old Blue environment to release resources:

1. Go to **EC2** -> **Auto Scaling Groups** -> Select `asg-otms-attendance-blue`.
2. Under the **Details** tab, click **Edit** on group details.
3. Update group sizes: Desired = `0`, Min = `0`, Max = `5` (or `0`).
4. Save the configuration.

AWS will terminate all instances in the Blue ASG. The rollout is complete.

---


## 6. Rollout and Rollback Strategy (AWS Console)

This section details how the Blue-Green strategy is achieved (Rollout) and how to revert changes in case of failure (Rollback) manually using the AWS Management Console.

### 6.1 Achieving the Rollout (Deployment Flow)
To deploy a new software version:
1. **Provision Green Environment**: Navigate to the EC2 Console and create the Green Target Group (`tg-otms-attendance-green`), then launch/configure the Green ASG running version `v2.0.0`.
2. **Verify Health**: Check the health status in the Green Target Group and ensure all instances show as `healthy`.
3. **Shift Traffic**: Edit the ALB Listener Rule for path `/api/v1/attendance*` and switch the forward-to Target Group from Blue to Green. Traffic routes instantly to `v2.0.0`.
4. **Clean up**: Edit the Blue ASG settings and scale the capacity down to `0` once Green is verified as stable.

### 6.2 Reverting the Rollout (Rollback Flow)
If the new Green environment fails health verification or shows production issues:
1. **Instant Rollback**: Edit the ALB Listener Rule immediately to switch the forward-to target group from Green back to `tg-otms-attendance-blue`. All user traffic reverts instantly to the stable Blue environment.
2. **Decommission**: Scale down the faulty Green ASG desired capacity to `0` to release resources and investigate the deployment failure.
   
---



## 6. Troubleshooting

| Issue                                                 | Cause                                                                               | Solution                                                                                            |
| ----------------------------------------------------- | ----------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------- |
| **New instances fail ALB Health Checks**              | The application is not running on port 8081, or the health-check path is incorrect. | Check the Instance User Data logs (`/var/log/cloud-init-output.log`) to verify application startup. |
| **ALB Listener Rule Priorities Conflict**             | A rule with priority `20` already exists on the target listener.                    | Change the listener rule priority in `main.tf` to a unique number.                                  |
| **Terraform State Lock Error**                        | A previous Terraform apply failed or did not release the lock.                      | Run `terraform force-unlock <LOCK_ID>` inside the Terraform directory.                              |
| **Instances scale up but traffic doesn't reach them** | The security group does not allow inbound port 8081 from the ALB.                   | Check that the target group security group matches the `dev-otms-attendance-sg` configuration.      |

---

## 7. FAQs

| Question                                 | Answer                                                                          |
| ---------------------------------------- | ------------------------------------------------------------------------------- |
| **What is Blue-Green Deployment?**       | It uses two environments: Blue is active and Green is used for the new version. |
| **Why do we use Blue-Green Deployment?** | To deploy with no downtime and allow quick rollback.                            |
| **How is the new version deployed?**     | The new version is first deployed to the inactive environment.                  |
| **How is the new environment checked?**  | The ALB checks the application health on port `8081`.                           |
| **How is traffic switched?**             | Terraform updates the ALB to send traffic from Blue to Green.                   |

---

## 8. Contact Information

| Name           | Email                                                                                 |
| -------------- | ------------------------------------------------------------------------------------- |
| Bhawna Dangarh | [bhawna.dangarh.snaatak@mygurukulam.co](mailto:bhawna.dangarh.snaatak@mygurukulam.co) |

---

## 9. References

| Description                                    | Link                                                                                                 |
| ---------------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| HashiCorp Terraform AWS Provider Documentation | https://registry.terraform.io/providers/hashicorp/aws/latest                                         |
| AWS Application Load Balancer Target Group     | https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html |
| Blue-Green Deployments on AWS                  | https://docs.aws.amazon.com/whitepapers/latest/blue-green-deployments/introduction.html              |
