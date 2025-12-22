# Sentasity AWS Assessment Templates

This repository contains the CloudFormation templates used by [Sentasity](https://sentasity.com) to perform secure, view-only cost optimization and security assessments.

These templates are designed with a **"Security First"** architecture:
* **View-Only Access:** We cannot modify, delete, or create resources in your account.
* **No Agents:** No software is installed on your EC2 instances or servers.
* **Audit Trail:** Every action performed by our role is logged in your AWS CloudTrail.
* **Zero-Trust Revocation:** You can remove our access instantly by deleting the CloudFormation stack.

---

## Templates

### 1. Standard Assessment (Default)
**File:** `sentasity-assessment-standard.yaml`

This is the recommended template for 95% of assessments. It provides the minimum permissions required to scan your infrastructure for cost savings (idle resources, rightsizing) and security vulnerabilities.

| Type | Permission | Justification |
| :--- | :--- | :--- |
| **Managed Policy** | `ViewOnlyAccess` | Allows viewing resource metadata (e.g., EC2 instance types, RDS sizes). |
| **Managed Policy** | `SecurityAudit` | Allows viewing security configurations (e.g., Security Groups, IAM password policies). |

### 2. Advanced Snapshot Analysis (Optional)
**File:** `sentasity-assessment-advanced.yaml`

Use this template if you require deep-dive analysis of EBS Snapshots to identify orphaned or incremental storage waste. This role includes the Standard permissions plus specific access to the EBS Direct APIs.

| Type | Permission | Justification |
| :--- | :--- | :--- |
| **Action** | `ebs:ListSnapshotBlocks` | Required to calculate the exact size of incremental snapshots to find "hidden" storage costs. |
| **Action** | `kms:Decrypt` | **Strictly scoped to EBS service only.** Required to read the *metadata* of encrypted snapshots. We cannot access the volume data itself, only the block structure for cost analysis. |

---

## Deployment Options

Our templates support two deployment modes via a single CloudFormation parameter (`DeploymentScope`).

### Mode A: Standalone (Single Account)
* **Use Case:** You want to scan a single AWS account.
* **Mechanism:** Creates one IAM Role in the current account.

### Mode B: Organization (Multi-Account)
* **Use Case:** You want to scan your Management (Payer) account and all Member accounts automatically.
* **Mechanism:**
    1.  Creates an IAM Role in the Management account.
    2.  Deploys a **StackSet** that automatically provisions the View-Only role to all existing (and future) member accounts in your Organization.
    3.  **Requirement:** You must have [Trusted Access enabled for StackSets](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-orgs-activate-trusted-access.html) in your Organization settings.

---

## Security FAQ

**Q: Who can assume these roles?**
A: The trust policy for these roles is strictly locked to Sentasity's shared services account (`886557787053`). No other AWS account can assume this role.

**Q: Can Sentasity see my sensitive data (S3 objects, Databases)?**
A: **No.** The `ViewOnlyAccess` policy does not grant permission to `s3:GetObject`, `dynamodb:GetItem`, or `secretsmanager:GetSecret`. We can see *that* a bucket exists and how large it is, but we cannot see what is inside it.

**Q: How do I remove access?**
A: Simply delete the CloudFormation stack named `sentasity-viewonly-role` (or `sentasity-org-scan`) in your AWS Console. Access is immediately revoked across all accounts.

---

## Support

If you have questions about these templates or the assessment process, please contact us:

* **Email:** support@sentasity.com
* **Web:** [sentasity.com](https://sentasity.com)
