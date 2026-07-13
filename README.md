# Sentasity AWS Access Templates

The CloudFormation templates you deploy to connect an AWS account to [Sentasity](https://sentasity.com) for cost optimization and security scanning.

These are the same templates served from our public S3 bucket (`https://sentasity-public-cf-templates.s3.us-east-1.amazonaws.com/v2/`) and linked from our [Trust Center](https://sentasity.com/trust). This repository exists so you can read every line — and its history — before deploying anything.

**Design principles:**

* **Read-only access.** The roles are built on AWS-managed read-only policies. We cannot start, stop, resize, terminate, or modify resources in your account.
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
| `SentasityReadOnlyRole` | Cost and usage data access | AWS-managed [`ReadOnlyAccess`](https://docs.aws.amazon.com/aws-managed-policy/latest/reference/ReadOnlyAccess.html), plus CUR discovery reads (`cur:DescribeReportDefinitions`, `bcm-data-exports:ListExports`/`GetExport`) |

Both roles trust `sts:AssumeRole` from Sentasity's shared services account (`886557787053`) only.

### 2. `SentasityAccessOrganization.yaml` — the same roles, org-wide

Deploys the same two roles into the management account and, via a `SERVICE_MANAGED` CloudFormation StackSet, into every member account of your AWS Organization (including accounts added later). Deploy in the management (payer) account. Requires [trusted access for StackSets](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-orgs-activate-trusted-access.html) — which the Org Bootstrap template below enables for you.

### 3. `SentasityOrgBootstrap.yaml` — organization discovery

A minimal role (`SentasityOrgDiscoveryRole`) for browsing your AWS Organizations structure — listing accounts and OUs so you can choose what to connect — plus a small custom resource that enables the AWS service access (CloudTrail, StackSets) the org-wide deployment needs. Deployed to the management (payer) account only. Read access only, except `organizations:EnableAWSServiceAccess` / `cloudformation:ActivateOrganizationsAccess` for that one-time enablement.

### 4. `SentasityCostUsageReport.yaml` — CUR delivery (optional)

Creates an S3 bucket and a Cost and Usage Report (CUR 1.0, Parquet) definition configured for Sentasity's cost analysis pipeline. Used by Spend Explorer. The only principal granted access to the bucket is AWS's own billing delivery service — Sentasity reads the reports through `SentasityReadOnlyRole`. Must be deployed in `us-east-1` (an AWS requirement for CUR report definitions).

---

## Security FAQ

**Q: Who can assume these roles?**
A: Only Sentasity's shared services account (`886557787053`). The trust policy on every role is locked to that single principal — no other AWS account can assume them.

**Q: Can Sentasity modify anything in my account?**
A: No. The roles carry AWS-managed read-only policies and supplemental statements limited to `Describe*`, `List*`, and `Get*` actions. The one exception is the Org Bootstrap's one-time service-access enablement described above — it enables AWS-side integration switches, and touches nothing else.

**Q: What data does Sentasity read?**
A: Billing and cost data, resource configuration metadata, and security posture — the signals needed to price waste and flag risk. See the [Trust Center](https://sentasity.com/trust) for exactly what we store, where it lives, and how tenants are isolated.

**Q: How do I remove access?**
A: Delete the CloudFormation stack (`Sentasity-Access`, and `Sentasity-Org-Bootstrap` if deployed) in your AWS Console. Access is revoked immediately; for organization deployments, deleting the stack removes the StackSet-deployed roles from member accounts as well.

---

## Versioning

The deployed templates are versioned under the S3 bucket prefix (currently `v2/`); this repository tracks the current version. Breaking changes get a new version prefix and old versions remain available forever, so a stack you deployed keeps working unchanged.

---

## Support

Questions about these templates or the connection process:

* **Email:** support@sentasity.com
* **Web:** [sentasity.com](https://sentasity.com) · [Trust Center](https://sentasity.com/trust)
