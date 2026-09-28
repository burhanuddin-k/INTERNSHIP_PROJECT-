# AWS Multi-Account Governance using AWS Organizations & Service Control Policies (SCPs)

**Institute Project — DevOps & Cloud Computing (AWS)**

## 📌 Project-1 Overview

This project demonstrates how to design and enforce **centralized governance** across a multi-account AWS environment using **AWS Organizations** and **Service Control Policies (SCPs)**.

Instead of relying on individual IAM policies inside every account, this project shows how a central **Management Account** can define guardrails that apply automatically to all member accounts (Dev, Prod, Test) — preventing risky, costly, or non-compliant actions **before** they ever reach IAM evaluation.

### Objectives
- Set up an AWS Organization with multiple Organizational Units (OUs).
- Create and manage member accounts under the correct OU.
- Author and attach custom **SCPs** to restrict specific actions.
- Validate that the guardrails actually block the intended actions.
- Document real console output/errors as proof of enforcement.

---

## 🏗️ Architecture

```
Root (Management Account)
 ├── DEV OU
 │     └── Dev-Account   ← EC2 restrictions, region lock
 ├── PROD OU
 │     └── (Production accounts)
 └── TEST OU
       └── (Test accounts)
```

- **Management Account** – Hosts AWS Organizations and owns all SCPs.
- **OUs (DEV / PROD / TEST)** – Logical grouping of accounts by environment.
- **SCPs** – Attached at the OU/account level to set maximum allowed permissions.

---

## 🛠️ Tools & Services Used

| Service | Purpose |
|---|---|
| AWS Organizations | Central multi-account management, OU structure |
| Service Control Policies (SCP) | Preventive guardrails (Deny-based policies) |
| AWS IAM | Role assumption (`OrganizationAccountAccessRole`) into member accounts |
| Amazon EC2 | Target service used to test instance-type & region restrictions |
| AWS CloudTrail | Target service used to test "protect logging" guardrail |

---

## 🚀 Steps Performed

### 1. Create the Organization Structure
Created a Root with three Organizational Units — **DEV**, **PROD**, and **TEST** — to logically separate environments.
<img width="1900" height="672" alt="2" src="https://github.com/user-attachments/assets/3f9df38d-2baf-4a29-b5a5-69b56b15d15b" />

### 2. Create a Member Account
Attempted to create a new **Dev-Account** under the DEV OU. A couple of early attempts failed due to a duplicate/root email conflict (`EMAIL_ALREADY_EXISTS`), which is a great real-world troubleshooting examing
<img width="1917" height="731" alt="3" src="https://github.com/user-attachments/assets/0335354d-f200-45bc-9dc5-94f7ebf76626" />

Account creation succeeded on a retry with a unique email, and the account was correctly placed inside the **DEV** OU.

<img width="1830" height="631" alt="4" src="https://github.com/user-attachments/assets/de5474bb-5a7b-4deb-a4cc-bc251ef61784" />


### 3. Author SCP #1 — Deny Large EC2 Instance Types (`DenyLargeEC2InDev`)
To control cost in the Dev environment, created an SCP that explicitly denies `ec2:RunInstances` for a list of large/expensive instance families (`m5.*`, `c5.*`, `r5.*` — 2xlarge and above).

<img width="698" height="807" alt="5" src="https://github.com/user-attachments/assets/4d8a269b-1c00-4474-b4bd-f5076b7a4adc" />

The policy JSON uses a `Deny` effect with a `StringLike` condition on `ec2:InstanceType`:

<img width="716" height="708" alt="6" src="https://github.com/user-attachments/assets/6e1ff9be-915d-45fc-9b08-cb62b608ff48" />

### 4. Attach SCP #1 to the Dev Account
Once created, the policy was attached directly to the **Dev-Account** (and to the DEV OU) alongside the default `FullAWSAccess` policy.

<img width="1542" height="727" alt="7" src="https://github.com/user-attachments/assets/b92592d7-6083-4dc1-ae8c-2dff2b4008a3" />


Confirmed via the account's **Policies** tab that 6 policies were now applied (directly, via DEV OU, and via Root).

<img width="1540" height="727" alt="8" src="https://github.com/user-attachments/assets/17eb99f8-00c7-45ae-9640-87523b0f23eb" />


Account details for reference:

![Dev-Account details](<img width="1440" height="746" alt="9" src="https://github.com/user-attachments/assets/9d7ad4bf-21a5-4ac5-9158-303f8e91b33b" />
)

### 5. Validate SCP #1 — Launch a Large EC2 Instance (Expected: Denied)
Logged into the Dev-Account console and attempted to launch a large EC2 instance.

<img width="1892" height="853" alt="10" src="https://github.com/user-attachments/assets/a183843c-3b48-4894-a994-7a4b4e4880e2" />


✅ **Result: Blocked as expected.** The console returned an explicit deny, referencing the SCP directly:

> `User: ...assumed-role/OrganizationAccountAccessRole/org-admin is not authorized to perform: ec2:RunInstances ... with an explicit deny in a service control policy: arn:aws:organizations::...:policy/o-s4obcjcl4x/service_control_policy/p-sc54wtt0`

<img width="1917" height="856" alt="11" src="https://github.com/user-attachments/assets/ce11da1c-6a03-4e5b-beb9-cbc762809509" />


### 6. Author SCP #2 — Protect CloudTrail (`ProtectCloudTrail`)
Created a second SCP to prevent any member account from stopping or deleting CloudTrail trails, protecting the org's audit logging.

<img width="1905" height="797" alt="12" src="https://github.com/user-attachments/assets/6a97df90-c35b-4d4d-8072-4b3e234568e2" />


Attached it at the **Root** level so it applies organization-wide.

<img width="1897" height="798" alt="Screenshot 2026-09-11 003007" src="https://github.com/user-attachments/assets/4855e9b0-29c7-4d08-b369-0433b39c0119" />


### 7. Author SCP #3 — Restrict AWS Regions (`RestrictAWSRegions`)
Created a third SCP, `RestrictAWSRegions`, to confine member accounts to a set of approved AWS regions.

<img width="1911" height="788" alt="Screenshot 2026-09-11 003257" src="https://github.com/user-attachments/assets/12539f28-8b97-48d7-9499-8d9f2375ef1e" />


Attached it at the **Root** level as well, so all four SCPs (`DenyLargeEC2InDev`, `DenyLeaveAndCloseAccount`, `ProtectCloudTrail`, `RestrictAWSRegions`) now govern the organization.

<img width="1915" height="791" alt="Screenshot 2026-09-11 003400" src="https://github.com/user-attachments/assets/25376490-8559-4eb2-bc10-88fb05fe64cc" />


### 8. Validate SCP #2 — Try to Stop CloudTrail Logging (Expected: Denied)
Created a test trail (`Governance-Test-Trail`) in the Dev-Account to verify the CloudTrail protection guardrail.

<img width="1917" height="867" alt="Screenshot 2026-09-11 005701" src="https://github.com/user-attachments/assets/c68e1945-a2c5-44d6-b748-ffaebeacedeb" />


Selected the trail to attempt to stop logging:

<img width="1917" height="688" alt="Screenshot 2026-09-11 005856" src="https://github.com/user-attachments/assets/853c7f96-6aee-4cd9-ab61-d635c4728fc5" />


✅ **Result: Blocked as expected.** Clicking **Stop logging** returned an `AccessDeniedException`, explicitly citing the SCP:

> `User: ...assumed-role/OrganizationAccountAccessRole/org-admin is not authorized to perform: cloudtrail:StopLogging ... with an explicit deny in a service control policy: arn:aws:organizations::...:policy/o-s4obcjcl4x/service_control_policy/p-ttv9c60o`

<img width="1896" height="860" alt="Screenshot 2026-09-11 010008" src="https://github.com/user-attachments/assets/a6000d7a-af38-4c63-bf8e-f6ec85443286" />


### 9. Validate SCP — Describe EC2 Instances (Expected: Denied)
Also confirmed that even a read-only action like `ec2:DescribeInstances` is blocked in the Dev-Account by an SCP, demonstrating how tightly the guardrails were scoped:

> `... is not authorized to perform: ec2:DescribeInstances with an explicit deny in a service control policy: arn:aws:organizations::...:policy/o-s4obcjcl4x/service_control_policy/p-vpm3d96n`

<img width="1895" height="807" alt="Screenshot 2026-09-11 010816" src="https://github.com/user-attachments/assets/2b1f09be-7339-4f20-9115-1bba4e194604" />

## 🎓 Conclusion
-------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Centralized CI/CD Platform Setup for Multiple Applications using Shared Jenkins Infrastructure

## Project 2 – Internship Project Set 8

## Objective

Build a centralized Jenkins CI/CD platform that supports multiple applications using:

- Jenkins on AWS EC2
- Jenkins Agents
- Jenkins Shared Library
- Standard Build, Test, Scan and Deploy stages
- Multiple applications
- Credential management
- Role-based access control

---

## 1. Jenkins Infrastructure

Jenkins is deployed on an AWS EC2 instance and is used as the central CI/CD server.

![Jenkins Controller EC2](screenshots/01-jenkins-controller-ec2.png)

A separate EC2 instance was created for the Jenkins agent.

![Jenkins Agent EC2](screenshots/02-jenkins-agent-ec2.png)

The agent environment contains Docker and Git.

![Agent Docker and Git](screenshots/20-agent-docker-git-environment.png)

The Jenkins node configuration is shown below.

![Jenkins Nodes](screenshots/21-jenkins-nodes.png)

---

## 2. Docker

Docker is installed on the Jenkins infrastructure and is used for the application container workflow.

![Docker Installation](screenshots/03-docker-controller.png)

Docker containers can be checked from the Jenkins environment.

![Docker Containers](screenshots/04-docker-container-check.png)

---

## 3. Jenkins Shared Library

A separate GitHub repository was created for the Jenkins Shared Library.

Repository:

`jenkins-shared-library`

![Jenkins Shared Library](screenshots/05-shared-library-repository.png)

The shared library is configured in Jenkins as a Global Trusted Pipeline Library.

![Global Shared Library Configuration](screenshots/10-global-shared-library-config.png)

The applications load the library using:

```groovy
@Library('jenkins-shared-library') _
