# AWS Blue/Green Deployment with Jenkins and Ansible

This project demonstrates a classic and robust Blue/Green deployment strategy on AWS, orchestrated by Jenkins and configured by Ansible. The goal is to achieve zero-downtime deployments with a manual approval gate for safety before switching live traffic.

## Overview

The architecture uses two persistent environments, "Blue" and "Green," each consisting of an EC2 instance. An Application Load Balancer (ALB) directs traffic to one of these environments at a time. The Jenkins pipeline automates the process of deploying new code to the inactive environment, running health checks, and then switching live traffic automatically after manual approval.

### Core Technologies

- **Cloud Provider:** Amazon Web Services (AWS)
- **CI/CD:** Jenkins
- **Configuration Management:** Ansible
- **Web Server:** Apache HTTPD
- **Key AWS Services:** EC2, Application Load Balancer (ALB), Target Groups, Security Groups

---

## Architecture Diagram

![image](https://github.com/user-attachments/assets/058b9f35-38a4-44d6-830a-31014071dfe7)

---

## Project Visualization

Here is a look at the key components of the project in action.

### Jenkins Pipeline

The heart of the automation, showing the stages of the deployment.

![Jenkins Pipeline](https://github.com/user-attachments/assets/4bb99551-bcd6-4a9d-b785-36f346ca014d)

### Blue & Green Environments

The live application view when traffic is directed to the respective server.

**_Blue Server Live_**  
![Blue Server](https://github.com/user-attachments/assets/fafe1555-dbee-49ea-a017-6f0d1cf7e352)

**_Green Server Live_**  
![Green Server](https://github.com/user-attachments/assets/41356bd5-2c7e-413c-afae-12a080f6c643)

### AWS Infrastructure

A look at the configured AWS resources that support the deployment.

**EC2 Instances**  
![EC2 Instances](https://github.com/user-attachments/assets/d2c567e6-3f9e-43df-b9a8-68bff999c441)

**Application Load Balancer**  
![Load Balancer](https://github.com/user-attachments/assets/8fc82558-52d4-4d65-978f-92c60008fc2c)

**Target Groups**  
![Target Groups](https://github.com/user-attachments/assets/3b60e37a-ef9c-4262-9144-67def8e3e6d9)

**Listener Rules**  
_The "dummy" health check rules are essential for ensuring the inactive environment is always being health-checked._  
![Listener Rules](https://github.com/user-attachments/assets/1c1ac1ae-86af-4f4b-afb0-f0f33775b5e3)

---

## Features

- **Zero-Downtime Deployments:** Users experience no interruption during the update process.
- **Persistent Environments:** Reuses the same Blue and Green EC2 instances for each deployment, simplifying the infrastructure.
- **Automated Configuration:** Ansible playbooks handle the installation and configuration of the web server (`httpd`).
- **Dynamic Inventory:** Ansible automatically discovers EC2 instances using AWS tags.
- **Safety Gates:** The Jenkins pipeline includes manual `input` steps for confirming deployment start and the final traffic switch, providing a crucial layer of safety.
- **Automated Health Checks:** Before the final approval, the pipeline automatically verifies that the newly deployed environment is healthy.
- **Automated Traffic Switch:** After approval, the pipeline uses the AWS CLI to atomically switch the ALB listener rule.

---

## Prerequisites

Before you can run this project, you need the following:

1. **An AWS Account** with an IAM user (`jenkins-user`) configured with `AdministratorAccess` (for simplicity).
2. **A Jenkins Server** with the following installed:
    - `ansible`
    - `git`
    - `jq`
    - `python3-boto3` (or `boto3` and `botocore` via pip)
    - `amazon.aws` Ansible collection (`ansible-galaxy collection install amazon.aws`)
3. **Jenkins Plugins Installed:**
    - `Ansible Plugin`
4. **Jenkins Configuration:**
    - Ansible tool configured in "Manage Jenkins" -> "Tools".
    - AWS credentials (`aws-access-key`, `aws-secret-key`) stored in the Jenkins Credentials Manager.

---

## Setup and Implementation

Refer the `Implementation_Guide.md` file for detailed Implementation Guide.

This project requires a one-time manual setup of the AWS infrastructure.

1. **Create Security Groups:**
    - `alb-sg`: Allows inbound HTTP (port 80) from anywhere.
    - `app-sg`: Allows inbound HTTP from `alb-sg` and inbound SSH (port 22) from your Jenkins server's IP.
2. **Launch EC2 Instances:**
    - Launch a `blue-server` and a `green-server` using an Amazon Linux AMI.
    - Assign the `app-sg` security group to both.
    - Tag them correctly: **Key:** `Deployment`, **Value:** `Blue` / `Green`.
3. **Create Target Groups:**
    - Create `blue-tg` and `green-tg`.
    - Register `blue-server` to `blue-tg` and `green-server` to `green-tg`.
4. **Create Application Load Balancer:**
    - Create `my-app-alb` and assign the `alb-sg` security group.
    - Set the default listener rule to forward traffic to `blue-tg`.
    - **Crucially:** Add two non-default listener rules (e.g., based on paths `/health-check-for-blue*` and `/health-check-for-green*`) that forward to `blue-tg` and `green-tg` respectively. This forces AWS to continuously health-check both environments.
5. **Place SSH Key on Jenkins Server:**
    - Copy your EC2 private key (`.pem` file) to the Jenkins server at `/var/lib/jenkins/YourKey.pem`.
    - Set ownership and permissions: `sudo chown jenkins:jenkins ...` and `sudo chmod 400 ...`.
6. **Update Project Files:**
    - Update `group_vars/all.yml` with the correct path to your SSH key.
    - Update the top of the `Jenkinsfile` with the correct ARNs for your ALB listener and target groups.

---

## How to Use

1. Create a new "Pipeline" job in Jenkins.
2. Configure the job to use "Pipeline script from SCM".
3. Point it to this Git repository.
4. Click **"Build Now"**.
5. The pipeline will run, pausing at two key moments for your manual input:
    - To confirm the start of the deployment.
    - For final approval before switching traffic to the newly verified healthy environment.

---

## Key Files Overview

| File                     | Purpose                                                                                              |
|--------------------------|------------------------------------------------------------------------------------------------------|
| **`Jenkinsfile`**        | The heart of the CI/CD process. Defines all stages, logic, and user interaction points.             |
| **`configure_blue.yml`** | Ansible playbook to install `httpd` and deploy the "blue" version of the app.                       |
| **`configure_green.yml`**| Ansible playbook to install `httpd` and deploy the "green" version of the app.                      |
| **`traffic_switch.yml`** | A simple Ansible playbook that runs locally on Jenkins to execute the AWS CLI command for switching the ALB listener rule. |
| **`aws_ec2.yml`**        | The Ansible dynamic inventory file. Tells Ansible how to connect to AWS and find hosts based on their tags. |
| **`group_vars/all.yml`** | Contains global Ansible variables, primarily used here to define the SSH user and private key file for connecting to EC2 instances. |
