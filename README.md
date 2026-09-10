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

![AWS Organizations OU structure](images/2.png)

### 2. Create a Member Account
Attempted to create a new **Dev-Account** under the DEV OU. A couple of early attempts failed due to a duplicate/root email conflict (`EMAIL_ALREADY_EXISTS`), which is a great real-world troubleshooting example.

![Failed account creation requests](images/3.png)

Account creation succeeded on a retry with a unique email, and the account was correctly placed inside the **DEV** OU.

![Dev-Account created under DEV OU](images/4.png)

### 3. Author SCP #1 — Deny Large EC2 Instance Types (`DenyLargeEC2InDev`)
To control cost in the Dev environment, created an SCP that explicitly denies `ec2:RunInstances` for a list of large/expensive instance families (`m5.*`, `c5.*`, `r5.*` — 2xlarge and above).

![Creating DenyLargeEC2InDev policy](images/5.png)

The policy JSON uses a `Deny` effect with a `StringLike` condition on `ec2:InstanceType`:

![DenyLargeEC2InDev policy JSON](images/6.png)

### 4. Attach SCP #1 to the Dev Account
Once created, the policy was attached directly to the **Dev-Account** (and to the DEV OU) alongside the default `FullAWSAccess` policy.

![Attaching DenyLargeEC2InDev](images/7.png)

Confirmed via the account's **Policies** tab that 6 policies were now applied (directly, via DEV OU, and via Root).

![Applied policies on Dev-Account](images/8.png)

Account details for reference:

![Dev-Account details](images/9.png)

### 5. Validate SCP #1 — Launch a Large EC2 Instance (Expected: Denied)
Logged into the Dev-Account console and attempted to launch a large EC2 instance.

![Dev-Account console home](images/10.png)

✅ **Result: Blocked as expected.** The console returned an explicit deny, referencing the SCP directly:

> `User: ...assumed-role/OrganizationAccountAccessRole/org-admin is not authorized to perform: ec2:RunInstances ... with an explicit deny in a service control policy: arn:aws:organizations::...:policy/o-s4obcjcl4x/service_control_policy/p-sc54wtt0`

![EC2 launch denied by SCP](images/11.png)

### 6. Author SCP #2 — Protect CloudTrail (`ProtectCloudTrail`)
Created a second SCP to prevent any member account from stopping or deleting CloudTrail trails, protecting the org's audit logging.

![ProtectCloudTrail policy created](images/12.png)

Attached it at the **Root** level so it applies organization-wide.

![Attach ProtectCloudTrail at Root](images/13.png)

### 7. Author SCP #3 — Restrict AWS Regions (`RestrictAWSRegions`)
Created a third SCP, `RestrictAWSRegions`, to confine member accounts to a set of approved AWS regions.

![RestrictAWSRegions policy created](images/14.png)

Attached it at the **Root** level as well, so all four SCPs (`DenyLargeEC2InDev`, `DenyLeaveAndCloseAccount`, `ProtectCloudTrail`, `RestrictAWSRegions`) now govern the organization.

![Attach RestrictAWSRegions at Root](images/15.png)

### 8. Validate SCP #2 — Try to Stop CloudTrail Logging (Expected: Denied)
Created a test trail (`Governance-Test-Trail`) in the Dev-Account to verify the CloudTrail protection guardrail.

![CloudTrail trail list](images/16.png)

Selected the trail to attempt to stop logging:

![Selecting Governance-Test-Trail](images/17.png)

✅ **Result: Blocked as expected.** Clicking **Stop logging** returned an `AccessDeniedException`, explicitly citing the SCP:

> `User: ...assumed-role/OrganizationAccountAccessRole/org-admin is not authorized to perform: cloudtrail:StopLogging ... with an explicit deny in a service control policy: arn:aws:organizations::...:policy/o-s4obcjcl4x/service_control_policy/p-ttv9c60o`

![CloudTrail stop logging denied](images/18.png)

### 9. Validate SCP — Describe EC2 Instances (Expected: Denied)
Also confirmed that even a read-only action like `ec2:DescribeInstances` is blocked in the Dev-Account by an SCP, demonstrating how tightly the guardrails were scoped:

> `... is not authorized to perform: ec2:DescribeInstances with an explicit deny in a service control policy: arn:aws:organizations::...:policy/o-s4obcjcl4x/service_control_policy/p-vpm3d96n`

![EC2 DescribeInstances denied by SCP](images/19.png)

---

## ✅ Results & Key Learnings

| Guardrail | SCP Name | Validation Result |
|---|---|---|
| Block large/expensive EC2 instance types in Dev | `DenyLargeEC2InDev` | ❌ `ec2:RunInstances` explicitly denied |
| Prevent accounts from leaving the org / self-closing | `DenyLeaveAndCloseAccount` | Attached at Root — protects org integrity |
| Prevent tampering with audit logs | `ProtectCloudTrail` | ❌ `cloudtrail:StopLogging` explicitly denied |
| Restrict usage to approved AWS regions | `RestrictAWSRegions` | Attached at Root — confines account activity |

**Key takeaways from this project:**
1. **SCPs are preventive, not permissive** — they never grant access; they only set the *maximum* boundary. IAM permissions still have to be granted separately.
2. **Explicit Deny always wins** — even an account's own Administrator/root-equivalent role (`OrganizationAccountAccessRole`) cannot override an SCP deny.
3. **Centralized governance scales** — a handful of policies attached at the Root/OU level enforced cost, security, and compliance guardrails across every current and future account in the DEV OU without touching each account individually.
4. **Error messages are actionable** — AWS surfaces the exact policy ARN responsible for a denial, which makes auditing and troubleshooting SCPs straightforward.

---

## 📂 Repository Structure

```
.
├── README.md
└── images/
    ├── 2.png   → OU structure (Root / DEV / PROD / TEST)
    ├── 3.png   → Failed account creation (EMAIL_ALREADY_EXISTS)
    ├── 4.png   → Dev-Account created under DEV OU
    ├── 5.png   → Creating DenyLargeEC2InDev policy
    ├── 6.png   → DenyLargeEC2InDev JSON
    ├── 7.png   → Attaching DenyLargeEC2InDev
    ├── 8.png   → Applied policies on Dev-Account
    ├── 9.png   → Dev-Account details
    ├── 10.png  → Dev-Account console home
    ├── 11.png  → EC2 launch denied by SCP
    ├── 12.png  → ProtectCloudTrail policy created
    ├── project-1.png → AWS Organizations landing page
    ├── Screenshot_2026-09-11_003007.png → Attach ProtectCloudTrail
    ├── Screenshot_2026-09-11_003257.png → RestrictAWSRegions created
    ├── Screenshot_2026-09-11_003400.png → Attach RestrictAWSRegions
    ├── Screenshot_2026-09-11_005701.png → CloudTrail trail list
    ├── Screenshot_2026-09-11_005856.png → Selecting test trail
    ├── Screenshot_2026-09-11_010008.png → Stop logging denied
    └── Screenshot_2026-09-11_010816.png → DescribeInstances denied
```

---

## 🎓 Conclusion

This project simulates a real-world **enterprise landing zone** governance pattern: a management account defining organization-wide guardrails via SCPs, with member accounts (Dev/Prod/Test) inheriting those restrictions automatically. It highlights core DevOps/Cloud principles of **least privilege**, **policy-as-code**, and **defense in depth** in a multi-account AWS environment.

*Submitted as part of the DevOps & AWS institute coursework.*
