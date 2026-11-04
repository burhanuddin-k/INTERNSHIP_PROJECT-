# Project 5 — Hybrid Infrastructure Monitoring with AWS CloudWatch

## Overview
Monitored an **on-premises Ubuntu Server** running in **Oracle VirtualBox** using **Amazon CloudWatch**. The CloudWatch Agent collected host metrics and published them to AWS.

## Architecture
```text
Windows 11 → VirtualBox → Ubuntu Server 24.04 → CloudWatch Agent → Amazon CloudWatch (eu-north-1)
```

## Tools
Oracle VirtualBox, Ubuntu Server 24.04 LTS, Windows PowerShell/SSH, AWS CLI v2, IAM, CloudWatch Agent, CloudWatch Metrics.

## Implementation

### 1. Install VirtualBox and create Ubuntu VM
Created `Project-5-Ubuntu-New` with 4 GB RAM, 2 CPUs, 25 GB disk and NAT networking.

![VirtualBox installed](images/virtualbox-installed.png)
![VM creation](images/virtualbox-vm-creation.png)
![VM in VirtualBox](images/virtualbox-ubuntu-vm.png)

### 2. Install Ubuntu Server
Installed Ubuntu Server 24.04 LTS and verified that the VM booted.

![VM running](images/ubuntu-vm-running.png)
![Ubuntu installation](images/ubuntu-server-installation.png)
![Ubuntu setup](images/ubuntu-setup.png)
![Installation completed](images/ubuntu-install-complete.png)

### 3. Prepare the server and SSH
```bash
lsb_release -a
ping -c 4 google.com
hostname -I
sudo apt update
sudo apt install -y curl wget unzip
sudo apt install -y openssh-server
sudo systemctl enable --now ssh
```
Configured VirtualBox NAT port forwarding (Windows port `2222` to Ubuntu port `22`), then connected from PowerShell:
```powershell
ssh -p 2222 vboxuser@127.0.0.1
```

### 4. Configure AWS IAM
Created IAM user `Project5-CloudWatch-User` with `CloudWatchAgentServerPolicy` and an access key for the on-premises host.

![IAM permissions](images/iam-user-permissions.png)
![IAM user created](images/iam-user-created.png)

**Security:** Do not upload access keys or credentials to GitHub.

### 5. Install AWS CLI and verify access
```bash
curl -o awscliv2.zip https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip
unzip awscliv2.zip
sudo ./aws/install
aws --version
aws configure --profile project5
aws sts get-caller-identity --profile project5
```
Used AWS region `eu-north-1`.

![AWS CLI installed](images/aws-cli-installed.png)
![AWS identity verified](images/aws-connection-verified.png)

### 6. Install CloudWatch Agent
```bash
wget https://amazoncloudwatch-agent-eu-north-1.s3.eu-north-1.amazonaws.com/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb
sudo dpkg -i amazon-cloudwatch-agent.deb
/opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a status
```

![CloudWatch Agent installed](images/cloudwatch-agent-installed.png)

### 7. Configure monitoring
```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
```
Selected Linux, **On-Premises**, host metrics, overall CPU (not per core), **60-second** collection, Basic configuration. Disabled StatsD, log collection, X-Ray, and SSM Parameter Store. Saved the configuration to `/opt/aws/amazon-cloudwatch-agent/bin/config.json` and configured the agent to use the `project5` credentials profile.

![Agent configuration](images/cloudwatch-agent-configuration.png)

### 8. Start and verify agent
```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m onPremise -s -c file:/opt/aws/amazon-cloudwatch-agent/bin/config.json
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a status
```
The agent reported `running` and `configured`.

![Agent running](images/cloudwatch-agent-running.png)

### 9. Verify CloudWatch metrics
In **CloudWatch → Metrics → Classic metrics** (Stockholm), the **CWAgent** namespace displayed **29 metrics**. The `mem_used_percent` graph confirmed that metrics were reaching AWS.

![CWAgent namespace](images/cloudwatch-metrics-namespace.png)
![Memory usage graph](images/cloudwatch-memory-graph.png)

## Results
- On-premises Ubuntu Server successfully integrated with AWS CloudWatch.
- AWS CLI and IAM authentication verified.
- CloudWatch Agent running and configured during testing.
- CWAgent namespace and memory usage graph verified.

## Cost control and cleanup
After verification, stopped the CloudWatch Agent and disabled automatic startup:
```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a stop
sudo systemctl disable amazon-cloudwatch-agent
```
CloudWatch custom metrics may incur charges. Check AWS Billing and deactivate unused IAM access keys.

## Conclusion
Successfully monitored an on-premises Ubuntu Server from AWS CloudWatch.
