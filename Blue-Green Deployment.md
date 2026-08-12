#  Immutable infra rollout Blue Green Deployment

<img width="400" height="300" alt="image" src="https://github.com/user-attachments/assets/5c6b1ab1-7359-42b4-8b77-f54b51ac00d4" />


## Author Information

| Author | Created | Version | Last Updated By | Last Edited On | L0 Reviewer | L1 Reviewer | L2 Reviewer |
|---|---|---|---|---|---|---|---|
| Bhawna Dangarh | 2026-08-09 | 1.1 | Bhawna Dangarh | 2026-08-11 | Sharvari Khamkar / Tina Bhatnagar | Aman Raj | Abhishek Dubey |

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
7. [Contact Information](#7-contact-information)
8. [References](#8-references)

---

## 1. Introduction

Continuous Delivery (CD) is used to deploy application releases automatically, safely, and reliably. Rather than updating live EC2 instances in-place (Mutable Infrastructure)—which risks configuration drift and downtime—we employ **Blue-Green Deployment** (Immutable Infrastructure). 

In this setup, we maintain two identical production-ready environments:
- **Blue**: Represents the current active production environment.
- **Green**: Represents the inactive environment where the new version is deployed and verified before shifting traffic.

If an issue occurs, rollback is instantaneous, achieved by pointing the load balancer rule back to the healthy Blue environment.

---

## 2. Objective

- Automate zero-downtime application deployments.
- Build infrastructure configurations using Terraform to control target groups, Auto Scaling Groups (ASGs), and Application Load Balancer (ALB) routing rules.
- Demonstrate a complete, self-healing CD rollout using an automated shell script wrapper.
- Configure health checks to guarantee that unhealthy application versions do not receive user traffic.
- Reduce resource costs by scaling down the inactive environment's capacity to zero after a successful rollout.

---

## 3. Blue-Green Infrastructure Flowchart

The flowchart below demonstrates the rollout steps and rollback checkpoints implemented in this POC:

<img width="1536" height="800" alt="image" src="https://github.com/user-attachments/assets/7889c9ca-bad6-437a-9cb1-b913d11755c1" />


---

## 4. Step-by-Step Implementation

### 4.1 Terraform Configuration Setup

The deployment reuses the network infrastructure provisioned during earlier sprints. It searches for resources dynamically using `data` block lookups instead of hardcoding subnet and VPC IDs:

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

Initially, the **Blue** environment is marked as active. It runs version `v1.0.0` of the application. The AWS Auto Scaling Group (`asg_blue`) hosts 2 healthy instances, while the **Green** environment has a desired capacity of `0`.

```hcl
# terraform.tfvars
active_color      = "blue"
blue_desired_capacity  = 2
green_desired_capacity = 0
blue_app_version  = "v1.0.0"
green_app_version = "v1.0.0"
```

Initialize and deploy the initial active state:
```bash
terraform -chdir=terraform init
terraform -chdir=terraform apply -auto-approve
```

---

### 4.3 Deploy Update to Inactive (Green) Environment

To initiate the rollout of version `v2.0.0`, the wrapper script triggers an update targeting the inactive environment (Green). It updates the application version metadata and scales the Green ASG capacity to 2:

```bash
# Update Green environment while keeping Blue environment active
terraform -chdir=terraform apply \
  -var="green_desired_capacity=2" \
  -var="green_app_version=v2.0.0" \
  -auto-approve
```

This creates a parallel set of instances running `v2.0.0` under the Green Target Group (`tg_green`), while all client requests are still routed to the Blue Target Group (`tg_blue`).

---

### 4.4 Verify Health Checks on Inactive Target Group

The Load Balancer performs health checks against the Green environment target group on port `8081` at the designated path:

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

If health checks fail, the deployment script triggers an automated rollback to scale the Green environment back down to `0`.

---

### 4.5 Traffic Shift Listener Switch

Once all Green target instances are verified as healthy, we update the active target group pointer on the Application Load Balancer (ALB) Listener Rule to route 100% of incoming user traffic to the Green Target Group:

```hcl
action {
  type             = "forward"
  target_group_arn = var.active_color == "blue" ? aws_lb_target_group.tg_blue.arn : aws_lb_target_group.tg_green.arn
  order            = 2
}
```

Shift traffic command execution:
```bash
terraform -chdir=terraform apply \
  -var="green_desired_capacity=2" \
  -var="green_app_version=v2.0.0" \
  -var="active_color=green" \
  -auto-approve
```

---

### 4.6 Scale Down and Terminate Old Environment

After traffic has successfully shifted to the Green environment and verified for stability, we perform the final stage of the rollout by scaling down the Blue environment capacity to `0` to optimize resource utilization:

```bash
terraform -chdir=terraform apply \
  -var="green_desired_capacity=2" \
  -var="green_app_version=v2.0.0" \
  -var="active_color=green" \
  -var="blue_desired_capacity=0" \
  -auto-approve
```

At this stage, the Green environment has fully replaced the Blue environment. The rollback window is closed, completing the immutable rollout.

---

## 5. Commands Used

| Command | Description |
| --- | --- |
| `terraform -chdir=terraform init` | Initialises the terraform configuration and downloads necessary providers. |
| `terraform -chdir=terraform plan` | Generates execution plans to review changes prior to applying them. |
| `terraform -chdir=terraform apply` | Applies the changes to AWS infrastructure (e.g. scaling environments, switching listener routing). |
| `terraform -chdir=terraform output` | Queries the current output state variables (e.g. `active_environment`). |
| `bash deploy.sh` | Executes the complete, automated Blue-Green rollout pipeline script. |
| `curl -I http://<ALB-DNS-URL>/api/v1/attendance/health` | Validates API HTTP status responses. |

---

## 6. Troubleshooting

| Issue | Cause | Solution |
| --- | --- | --- |
| **New instances fail ALB Health Checks** | Application on port 8081 is not running, or path mismatch. | Check the Instance User Data logs (`/var/log/cloud-init-output.log`) on the server to verify application startup. |
| **ALB Listener Rule Priorities Conflict** | A rule with priority `20` already exists on the target listener. | Update the listener rule priority in `main.tf` to a unique number. |
| **Terraform State Lock Error** | A previous Terraform apply crashed or did not release lock. | Run `terraform force-unlock <LOCK_ID>` inside the terraform directory. |
| **Instances scale up but traffic doesn't reach them** | Security group lacks inbound port 8081 permission from the ALB. | Verify the target group security group matches the `dev-otms-attendance-sg` specifications. |

---

## 7. Contact Information

| Name | Email |
|------|-------|
| Bhawna Dangarh | [bhawna.dangarh.snaatak@mygurukulam.co](mailto:bhawna.dangarh.snaatak@mygurukulam.co) |

---

## 8. References

| Description | Link |
|------------|------|
| HashiCorp Terraform AWS Provider Documentation | [https://registry.terraform.io/providers/hashicorp/aws/latest](https://registry.terraform.io/providers/hashicorp/aws/latest) |
| AWS Application Load Balancer Target Group | [https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html) |
| Blue-Green Deployments on AWS | [https://docs.aws.amazon.com/whitepapers/latest/blue-green-deployments/introduction.html](https://docs.aws.amazon.com/whitepapers/latest/blue-green-deployments/introduction.html) |
