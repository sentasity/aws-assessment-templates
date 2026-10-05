# Sentasity AWS Access Templates

The CloudFormation templates you deploy to connect an AWS account to [Sentasity](https://sentasity.com) for cost optimization and security scanning.

These are the same templates served from our public S3 bucket (`https://sentasity-public-cf-templates.s3.us-east-1.amazonaws.com/v3/`) and linked from our [Trust Center](https://sentasity.com/trust). This repository exists so you can read every line — and its history — before deploying anything.

**Design principles:**

* **Read-only access.** The access roles are built on AWS-managed read-only policies. We cannot start, stop, resize, terminate, or modify resources in your account. (The optional Managed Services template, below, is the one exception, and only if you deploy it.)
* **Your own External ID.** Every role Sentasity assumes requires the External ID Sentasity issued to your organization, so knowing your account ID is never enough to reach it.
* **No agents.** Nothing is installed on your instances or servers.
* **Audit trail.** Every action our roles perform is logged in your AWS CloudTrail.
* **One-click revocation.** Delete the CloudFormation stack and our access ends immediately — the role is the only path in.

---

## Templates

### 1. `SentasityAccessStandalone.yaml` — the access roles, single account

The standard template for connecting one AWS account. Creates two cross-account IAM roles:

| Role | Purpose | Policies |
| :--- | :--- | :--- |
| `SentasityViewOnlySecurityRole` | Cost optimization and security scanning | AWS-managed [`ViewOnlyAccess`](https://docs.aws.amazon.com/aws-managed-policy/latest/reference/ViewOnlyAccess.html) + [`SecurityAudit`](https://docs.aws.amazon.com/aws-managed-policy/latest/reference/SecurityAudit.html), plus a supplemental statement of additional reads (`Describe*`, `List*`, `Get*` actions only) |
| `SentasityReadOnlyRole` | Cost and usage data access | AWS-managed [`ReadOnlyAccess`](https://docs.aws.amazon.com/aws-managed-policy/latest/reference/ReadOnlyAccess.html), plus explicit reads (`Get*`, `List*`, `Describe*`) for AWS billing and cost services (Cost Explorer, Cost and Usage Reports, credits, Savings Plans, budgets), and the permission to turn on [AWS Resource Explorer](https://docs.aws.amazon.com/resource-explorer/latest/userguide/welcome.html), AWS's free resource inventory. An explicit `Deny` blocks data reads: secret values, SSM parameters, database items, queue and stream messages, EC2 passwords and consoles, source code, and container images. |

Both roles trust `sts:AssumeRole` from Sentasity's shared services account (`886557787053`) only, and only with your Sentasity External ID (the `SentasityExternalId` parameter, which Sentasity's setup link fills in).

### 2. `SentasityAccessOrganization.yaml` — the same roles, org-wide

Deploys the same two roles into the management account and, via a `SERVICE_MANAGED` CloudFormation StackSet, into every member account of your AWS Organization (including accounts added later). The StackSet passes your External ID to every member account's roles. Deploy in the management (payer) account. Requires [trusted access for StackSets](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-orgs-activate-trusted-access.html) — which the Org Bootstrap template below enables for you.

### 3. `SentasityOrgBootstrap.yaml` — organization discovery

A minimal role (`SentasityOrgDiscoveryRole`, trusted with your External ID like the access roles) for browsing your AWS Organizations structure — listing accounts and OUs so you can choose what to connect — plus a small custom resource that enables the AWS service access (CloudTrail, StackSets) the org-wide deployment needs. Deployed to the management (payer) account only. Read access only, except `organizations:EnableAWSServiceAccess` / `cloudformation:ActivateOrganizationsAccess` for that one-time enablement.

### 4. `SentasityCostUsageReport.yaml` — CUR delivery (optional)

Creates an S3 bucket and a Cost and Usage Report (CUR 1.0, Parquet) definition configured for Sentasity's cost analysis pipeline. Used by Spend Explorer. The only principal granted access to the bucket is AWS's own billing delivery service — Sentasity reads the reports through `SentasityReadOnlyRole`. Must be deployed in `us-east-1` (an AWS requirement for CUR report definitions).

### 5. `SentasityCostUsageReport_BillingTransfer.yaml` — CURs for a billing transfer customer (optional)

For a bill transfer account that pays for other organizations through AWS Billing Transfer. Creates the same kind of S3 bucket and CUR 1.0 reports as template 4, one set per transferred customer, from that customer's billing views. Deploy in `us-east-1` from the bill transfer account, once per customer. Like template 4, it creates no role and grants Sentasity nothing new.

### 6. `SentasityManagedServicesAccess.yaml` — managed services roles (optional)

Only for customers who buy Sentasity's managed services and want Sentasity to troubleshoot and fix things in their account. Unlike the templates above, these roles **can change resources**: `SentasityPowerUserRole` carries AWS-managed [`PowerUserAccess`](https://docs.aws.amazon.com/aws-managed-policy/latest/reference/PowerUserAccess.html), and `SentasityAdminRole` (created only when `EnableAdminRole` is `true`) carries [`AdministratorAccess`](https://docs.aws.amazon.com/aws-managed-policy/latest/reference/AdministratorAccess.html). Both trust Sentasity's shared services account (`886557787053`) only, with your External ID when you provide one. Delete the stack and both roles are gone.

---

## Security FAQ

**Q: Who can assume these roles?**
A: Only Sentasity's shared services account (`886557787053`), and only when it presents the External ID Sentasity issued to your organization. The trust policy on every role is locked to that single principal and requires that External ID, so no other AWS account can assume them, and Sentasity cannot be tricked into assuming them on someone else's behalf ([the confused deputy problem](https://docs.aws.amazon.com/IAM/latest/UserGuide/confused-deputy.html)).

**Q: Can Sentasity modify anything in my account?**
A: Not through the access templates (1 to 5). Their roles carry AWS-managed read-only policies and supplemental statements limited to `Describe*`, `List*`, and `Get*` actions, with two narrow exceptions described above: the Org Bootstrap's one-time service-access enablement, which flips AWS-side integration switches and touches nothing else, and turning on AWS Resource Explorer. The Managed Services template (6) is the only one whose roles can modify resources, and it exists only if you deploy it.

**Q: What data does Sentasity read?**
A: Billing and cost data, resource configuration metadata, and security posture — the signals needed to price waste and flag risk. See the [Trust Center](https://sentasity.com/trust) for exactly what we store, where it lives, and how tenants are isolated.

**Q: How do I remove access?**
A: Delete the CloudFormation stack (`Sentasity-Access`, and `Sentasity-Org-Bootstrap` if deployed) in your AWS Console. Access is revoked immediately; for organization deployments, deleting the stack removes the StackSet-deployed roles from member accounts as well.

---

## Versioning

The deployed templates are versioned under the S3 bucket prefix (currently `v3/`); this repository tracks the current version. Breaking changes get a new version prefix and old versions remain available forever, so a stack you deployed keeps working unchanged.

---

## Support

Questions about these templates or the connection process:

* **Email:** support@sentasity.com
* **Web:** [sentasity.com](https://sentasity.com) · [Trust Center](https://sentasity.com/trust)
