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

## 4. Step-by-Step Implementation

### 4.1 Terraform Configuration Setup

This deployment uses the network infrastructure created in the earlier sprints.

Instead of directly writing VPC and subnet IDs, Terraform finds the required resources using `data` blocks:

```hcl
data "aws_vpc" "main" {
  tags = { Name = "${var.environment}-otms-vpc" }
}

data "aws_subnet" "backend" {
  tags = { Name = "${var.environment}_otms_backend_subnet_a" }
}

data "aws_security_group" "attendance" {
  tags = { Name = "${var.environment}-otms-attendance-sg" }
}

data "aws_lb" "otms_alb" {
  name = "${var.environment}-otms-alb"
}
```

---

### 4.2 Provision Active (Blue) Environment

Initially, the **Blue** environment is active.

It runs application version `v1.0.0`.

The Blue Auto Scaling Group (`asg_blue`) runs 2 instances, while the Green environment has a desired capacity of `0`.

```hcl
# terraform.tfvars
active_color      = "blue"
blue_desired_capacity  = 2
green_desired_capacity = 0
blue_app_version  = "v1.0.0"
green_app_version = "v1.0.0"
```

Initialize and deploy the initial environment:

```bash
terraform -chdir=terraform init
terraform -chdir=terraform apply -auto-approve
```

---

### 4.3 Deploy Update to Inactive (Green) Environment

To deploy version `v2.0.0`, the deployment script updates the inactive Green environment.

The Green ASG is scaled up to 2 instances and the new application version is deployed:

```bash
# Update Green environment while keeping Blue environment active
terraform -chdir=terraform apply \
  -var="green_desired_capacity=2" \
  -var="green_app_version=v2.0.0" \
  -auto-approve
```

This creates new instances running `v2.0.0` under the Green Target Group (`tg_green`).

At this time, users are still sending requests to the Blue Target Group (`tg_blue`).

---

### 4.4 Verify Health Checks on Inactive Target Group

The Load Balancer checks the health of the Green environment on port `8081`.

It uses the following health-check path:

```hcl
resource "aws_lb_target_group" "tg_green" {
  name     = "tg-otms-attendance-green"
  port     = 8081
  protocol = "HTTP"
  vpc_id   = data.aws_vpc.main.id

  health_check {
    path                = "/api/v1/attendance/health"
    port                = "8081"
    protocol            = "HTTP"
    matcher             = "200-399"
    interval            = 15
    timeout             = 5
    healthy_threshold   = 2
    unhealthy_threshold = 3
  }
}
```

If the health check fails, the deployment script rolls back by scaling the Green environment down to `0`.

---

### 4.5 Traffic Shift Listener Switch

After all Green instances pass the health checks, the Application Load Balancer (ALB) is updated to send 100% of user traffic to the Green Target Group.

```hcl
action {
  type             = "forward"
  target_group_arn = var.active_color == "blue" ? aws_lb_target_group.tg_blue.arn : aws_lb_target_group.tg_green.arn
  order            = 2
}
```

Traffic is switched using the following command:

```bash
terraform -chdir=terraform apply \
  -var="green_desired_capacity=2" \
  -var="green_app_version=v2.0.0" \
  -var="active_color=green" \
  -auto-approve
```

---

### 4.6 Scale Down and Terminate Old Environment

After the Green environment receives traffic successfully and remains stable, the old Blue environment is scaled down.

The Blue capacity is changed to `0`:

```bash
terraform -chdir=terraform apply \
  -var="green_desired_capacity=2" \
  -var="green_app_version=v2.0.0" \
  -var="active_color=green" \
  -var="blue_desired_capacity=0" \
  -auto-approve
```

Now the Green environment has completely replaced the Blue environment.

The rollback window is closed and the immutable rollout is complete.

---

## 5. Commands Used

| Command                                                 | Description                                                                                       |
| ------------------------------------------------------- | ------------------------------------------------------------------------------------------------- |
| `terraform -chdir=terraform init`                       | Initializes Terraform and downloads the required providers.                                       |
| `terraform -chdir=terraform plan`                       | Shows the changes Terraform will make before applying them.                                       |
| `terraform -chdir=terraform apply`                      | Applies the changes to AWS infrastructure, such as scaling environments and changing ALB routing. |
| `terraform -chdir=terraform output`                     | Shows the current Terraform output values, such as `active_environment`.                          |
| `bash deploy.sh`                                        | Runs the complete Blue-Green deployment script.                                                   |
| `curl -I http://<ALB-DNS-URL>/api/v1/attendance/health` | Checks whether the API is responding correctly.                                                   |

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
