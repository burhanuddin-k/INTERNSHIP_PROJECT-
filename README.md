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
<img width="1917" height="848" alt="project -2_1" src="https://github.com/user-attachments/assets/5428f348-3638-4607-915e-cedde7c6f566" />

### 2. Create a Member Account
Attempted to create a new **Dev-Account** under the DEV OU. A couple of early attempts failed due to a duplicate/root email conflict (`EMAIL_ALREADY_EXISTS`), which is a great real-world troubleshooting examing
<img width="1917" height="917" alt="ss-7" src="https://github.com/user-attachments/assets/def68876-6611-499b-86de-2102e20693de" />


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

In this project I set up Jenkins on an AWS EC2 instance and created a separate GitHub repository for a Jenkins Shared Library. I added two sample applications (`sample-app-1` and `sample-app-2`) that both use the same shared library in their Jenkinsfiles. I also configured GitHub webhooks and role-based access with Admin and Developer roles.

---

## 1. Jenkins Infrastructure

### Jenkins controller

Jenkins runs on an EC2 instance named `Project-2` in the Europe (Stockholm) region. The instance type is `c7i-flex.large` and it is in the Running state. The Jenkins page in my browser uses `13.48.196.145:8080`, which is the public IP of this instance.

![Jenkins controller EC2 instance](images/ec2-jenkins-controller.png)

This is the Jenkins home page after setup:

![Jenkins welcome page](images/jenkins-welcome-page.png)

### Jenkins agent EC2

I created a second EC2 instance named `project-2-jenkins-agent` for the agent. The instance type is `t3.small` and it is in the Running state.

![Jenkins agent EC2 instance](images/ec2-jenkins-agent.png)

On this instance I checked that Docker and Git are installed. `docker --version` shows Docker version 29.1.3, `git --version` shows git version 2.53.0, and `docker ps` runs without errors (no containers running).

![Docker and Git on the agent](images/agent-docker-git-version.png)

## Jenkins Node

A separate Jenkins node was configured on an AWS EC2 instance to support the Jenkins infrastructure.


![Jenkins Nodes page](images/jenkins-nodes-agent-na.png)

The node is configured with the required connection and execution settings in Jenkins.
---

## 2. Docker

Docker is installed on the Jenkins controller instance (`ip-172-31-11-248`). These are the steps visible in my terminal:

- `sudo systemctl enable docker` and `sudo systemctl start docker`
- `docker --version` shows Docker version 29.1.3
- `docker ps` first gave a permission denied error on `/var/run/docker.sock`
- I ran `sudo usermod -aG docker jenkins`, but `docker ps` still gave the same error in that session
- `sudo systemctl status docker` shows `docker.service` as active (running)

![Docker install and permission error on controller](images/docker-install-controller.png)

Then I ran `sudo usermod -aG docker $USER` and `newgrp docker`. After that `docker ps` worked and showed an empty container list.

![Docker permission fixed](images/docker-permission-fix.png)

The `docker image prune -f || true` command that shows up in the pipeline post actions (section 7) also uses Docker.

---

## 3. Jenkins Shared Library

I created a separate private GitHub repository called `jenkins-shared-library` (branch `main`). It has a `vars` folder and a `README.md`. The latest commit message is "Add standardized shared pipeline". The README in the repo says it is the centralized Jenkins shared library for Internship Project 2 and that every application uses the same four stages. 

![Jenkins shared library GitHub repository](images/github-jenkins-shared-library.png)

The library is added in Jenkins under Manage Jenkins → System. The screenshot shows:

- Retrieval method: Modern SCM
- Source Code Management: Git
- Project Repository: `https://github.com/burhanuddin-k/jenkins-shared-library.git`
- Credentials: `burhanuddin-k/****** (project-2)`
- Behaviours: Discover branches
- "Allow default version to be overridden" and "Include @Library changes in job recent changes" are checked
- "Load implicitly" is not checked
- It currently maps to revision `e7e50a0f09293cb96aa2bf9ff6aa8a58a847d133`, which matches the `e7e50a0` commit shown in the GitHub repo

The top of this section is cut off in my screenshot, so the library name field and the "Global Trusted Pipeline Libraries" heading are not visible in it. The library name used in the Jenkinsfiles is `jenkins-shared-library`.

![Global Pipeline Library configuration in Jenkins](images/jenkins-global-library-config.png)

Both applications call the shared pipeline like this:

```groovy
@Library('jenkins-shared-library') _

standardPipeline(
    appName: '...',
    testCommand: '...',
    dockerPort: ...,
    containerPort: ...
)
```

The stages shown in the successful pipeline runs (section 7) are Build, Test, Scan and Deploy. There is also a Post Actions stage after them. I did not open the files inside the `vars` folder in my screenshots, so the code of the shared library is not shown here.

---

## 4. Application 1

The repository is `sample-app-1` (private, branch `main`). According to its README it is a sample Flask application. The files are `Dockerfile`, `Jenkinsfile`, `README.md`, `app.py`, `requirements.txt` and `test_app.py`.

![sample-app-1 GitHub repository](images/github-sample-app-1-repo.png)

Jenkinsfile:

```groovy
@Library('jenkins-shared-library') _

standardPipeline(
    appName: 'sample-app-1',
    testCommand: 'pytest -q',
    dockerPort: 5001,
    containerPort: 5000
)
```

![sample-app-1 Jenkinsfile](images/sample-app-1-jenkinsfile.png)

The Jenkinsfile loads `jenkins-shared-library` and calls `standardPipeline` with the app name, test command and ports. The test command is `pytest -q`.

---

## 5. Application 2

The repository is `sample-app-2` (private, branch `main`). According to its README it is a sample Node.js application. The files are `Dockerfile`, `Jenkinsfile`, `README.md`, `app.js`, `app.test.js` and `package.json`.

![sample-app-2 GitHub repository](images/github-sample-app-2-repo.png)

Jenkinsfile:

```groovy
@Library('jenkins-shared-library') _

standardPipeline(
    appName: 'sample-app-2',
    testCommand: 'npm test',
    dockerPort: 5002,
    containerPort: 3000
)
```

![sample-app-2 Jenkinsfile](images/sample-app-2-jenkinsfile.png)

This Jenkinsfile loads the same `jenkins-shared-library` and calls the same `standardPipeline`. Only the values are different (app name, `npm test`, and the ports).

---

## 6. Multi-Application Support

Both applications use the same shared library and the same `standardPipeline` call. Only the parameters change.

| Application | Shared Library | Test Command | dockerPort | containerPort |
|---|---|---|---|---|
| sample-app-1 | jenkins-shared-library | `pytest -q` | 5001 | 5000 |
| sample-app-2 | jenkins-shared-library | `npm test` | 5002 | 3000 |

Pipeline structure (as seen in both pipeline runs in section 7):

Build → Test → Scan → Deploy

In both runs all four stages are green, followed by a Post Actions stage that is also green.

---

## 7. Jenkins Pipelines

### Jenkins dashboard

The dashboard shows both jobs, `sample-app-1` and `sample-app-2`. `sample-app-1` has last success #2 and last failure #1, so build #1 failed before #2 passed. `sample-app-2` has last success #3 and last failure N/A.

![Jenkins dashboard with both jobs](images/jenkins-dashboard-both-jobs.png)

### sample-app-1

Build #2 of `sample-app-1` was started by Admin and took 18 sec. The stage view shows Build (10s), Test (0.79s), Scan (0.32s) and Deploy (0.31s) all successful, then Post Actions (0.37s). The Post Actions stage ran `docker image prune -f || true` and printed "Standard pipeline completed successfully for sample-app-1."

![sample-app-1 pipeline run #2](images/jenkins-pipeline-sample-app-1.png)

### sample-app-2

Build #3 of `sample-app-2` was started by Admin and took 8.8 sec. The stage view shows Build (3s), Test (1s), Scan (0.3s) and Deploy (0.55s) all successful, then Post Actions (0.33s). It printed "Standard pipeline completed successfully for sample-app-2."

![sample-app-2 pipeline run #3](images/jenkins-pipeline-sample-app-2.png)

Both runs show "Started by Admin", so these were started from Jenkins by the Admin user.

---

## 8. GitHub Webhooks

I added a webhook in the settings of both `sample-app-1` and `sample-app-2`. In both repos the webhook points to `http://13.48.196.145:8080/github-w...` (the URL is cut off in the screenshot) and is set for the `push` event. GitHub shows "Last delivery was successful." for both. In the `sample-app-1` screenshot GitHub also shows the message that the hook was created and a ping was sent.

![sample-app-1 webhook](images/github-webhook-sample-app-1.png)

![sample-app-2 webhook](images/github-webhook-sample-app-2.png)

---

## 9. Credential Management

In the shared library configuration (section 3), the Credentials field is set to `burhanuddin-k/****** (project-2)`. The secret is masked in the screenshot. This credential is used by Jenkins when it fetches `jenkins-shared-library` from GitHub (the repository is private).

![Credentials selected in shared library configuration](images/jenkins-library-credentials.png)

---

## 10. Role-Based Access Control

### Security configuration

In Manage Jenkins → Security, the Security Realm is "Jenkins' own user database" with "Allow users to sign up" unchecked. Authorization is set to "Role-Based Strategy".

![Jenkins security configuration](images/jenkins-security-role-based.png)

### Users

There are 2 users in Jenkins' user database:

| Username | Full name |
|---|---|
| Develepor | Developer |
| project-2 | Admin |

The username is spelled `Develepor` in Jenkins (that is how it appears in the screenshot).

![Jenkins users](images/jenkins-users.png)

### Roles

Under Role Management → Manage Roles, two global roles exist:

| Role | Permissions shown |
|---|---|
| admin | Overall/Administer |
| Developer | Agent/Build, Job/Workspace, Overall/Read |

![Jenkins global roles](images/jenkins-global-roles.png)

### Role assignments

Role assignment page with only the `admin` role created.
![Role assignment with admin role](images/jenkins-role-assignment-admin.png)

Role assignment page after the `Developer` role was added.

![Role assignment with Developer and admin roles](images/jenkins-role-assignment-developer-admin.png)

---

## 11. Project Deliverables

- Jenkins Shared Library: `jenkins-shared-library` GitHub repository, configured in Jenkins
- Application 1: `sample-app-1`
- Application 2: `sample-app-2`
- Jenkins pipelines: `sample-app-1` (#2) and `sample-app-2` (#3), both with Build, Test, Scan and Deploy green
- Role-based access: Admin and Developer roles (Role-Based Strategy)
- GitHub integration: private repos, shared library over Git, push webhooks on both apps
- README (this file)

---

## Conclusion
