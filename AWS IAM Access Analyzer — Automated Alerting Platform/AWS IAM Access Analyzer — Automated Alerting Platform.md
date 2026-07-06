# AWS IAM Access Analyzer — Automated Alerting Platform

A production Terraform project deploying three complementary IAM Access Analyzer modules with automated Slack alerting for a multi-account AWS Organization (~70 accounts).

## Project Summary

| | External Access | Internal Access | Unused Access |
|--|----------------|-----------------|---------------|
| **Analyzer Type** | ORGANIZATION (free tier) | ORGANIZATION_INTERNAL_ACCESS | ORGANIZATION_UNUSED_ACCESS |
| **What it detects** | Resources shared publicly or outside the org | Resources shared cross-account within the org | Stale IAM roles, keys, passwords, permissions |
| **Severity filter** | HIGH + CRITICAL → Slack | CRITICAL only → Slack | CRITICAL only → Slack |
| **Account model** | All org accounts (via Security Hub) | Opt-in inclusion list | Opt-in (simulated via inverted exclusion) |
| **Cost** | Free | ~$0.20/resource/month | ~$0.20/IAM entity/month |
| **AWS Services** | Security Hub, EventBridge, Lambda, Secrets Manager | Access Analyzer, Security Hub, EventBridge, Lambda | Access Analyzer, Security Hub, EventBridge, Lambda |

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        AWS Organization (~70 accounts)               │
│                                                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐                          │
│  │ Account A│  │ Account B│  │ Account C│  ...                      │
│  └─────┬────┘  └─────┬────┘  └─────┬────┘                          │
│        └──────────────┼──────────────┘                               │
│                       │  findings                                    │
│                       ▼                                              │
│  ┌─────────────────────────────────────────────────────────────┐    │
│  │              Security Account (Delegated Admin)              │    │
│  │                                                             │    │
│  │  ┌─────────────────┐  ┌───────────────┐  ┌──────────────┐ │    │
│  │  │ External Access  │  │Internal Access│  │ Unused Access │ │    │
│  │  │ Analyzer (Free)  │  │ Analyzer      │  │ Analyzer     │ │    │
│  │  └────────┬─────────┘  └───────┬───────┘  └──────┬───────┘ │    │
│  │           │                     │                  │         │    │
│  │           ▼                     ▼                  ▼         │    │
│  │  ┌─────────────────────────────────────────────────────┐    │    │
│  │  │                   Security Hub                       │    │    │
│  │  │              (Centralized Aggregation)               │    │    │
│  │  └────────────────────────┬────────────────────────────┘    │    │
│  │                           │                                  │    │
│  │           ┌───────────────┼───────────────┐                 │    │
│  │           ▼               ▼               ▼                 │    │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐        │    │
│  │  │ EventBridge  │ │ EventBridge  │ │ EventBridge  │        │    │
│  │  │ (HIGH + CRIT)│ │ (CRIT only)  │ │ (CRIT only)  │        │    │
│  │  └──────┬───────┘ └──────┬───────┘ └──────┬───────┘        │    │
│  │         ▼                ▼                ▼                  │    │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐        │    │
│  │  │   Lambda     │ │   Lambda     │ │   Lambda     │        │    │
│  │  └──────┬───────┘ └──────┬───────┘ └──────┬───────┘        │    │
│  │         └────────────────┼────────────────┘                 │    │
│  │                          ▼                                   │    │
│  │                 ┌─────────────────┐                          │    │
│  │                 │ Secrets Manager  │                          │    │
│  │                 │ (Shared Webhook) │                          │    │
│  │                 └────────┬────────┘                           │    │
│  └──────────────────────────┼────────────────────────────────────┘    │
└─────────────────────────────┼─────────────────────────────────────────┘
                              ▼
                     ┌─────────────────┐
                     │  Slack Channel   │
                     └─────────────────┘
```

## Key Design Decisions

### 1. Shared Secret Pattern
A single Secrets Manager secret (owned by the external access module) is shared across all three Lambda functions. This avoids secret sprawl, reduces cost ($0.40/month for one secret vs $1.20 for three), and makes webhook rotation a single operation.

### 2. Account Opt-In Model
Both paid analyzers use opt-in — you explicitly list which accounts to scan. This controls cost and allows gradual rollout across the organization.

### 3. Inverted Exclusion Logic (Unused Access)
AWS only supports `exclusion` rules for unused access analyzers. Instead of manually maintaining a long exclusion list, the module:
1. Pulls all org accounts via the AWS Organizations API
2. Subtracts the opt-in list
3. Automatically excludes the analyzer owner account (AWS requirement)
4. Sorts the result to prevent ordering-related drift

This gives you a simple inclusion-style interface while handling the AWS API complexity internally.

### 4. Severity Filtering at EventBridge Level
Filtering is done in EventBridge event patterns (not Lambda) where possible. This reduces Lambda invocations and cost — the function only runs for findings that matter.

### 5. Conditional Resource Creation
Analyzers are only created when `enabled_accounts` is non-empty. If no accounts are configured, no analyzer is deployed and cost is $0.

### 6. Sorted Exclusion Lists
The exclusion list for the unused access analyzer is wrapped in `sort()` to produce a deterministic order. Without this, AWS returns org accounts in random order on each API call, causing Terraform to see a "change" and force an unnecessary (and expensive) analyzer replacement.

## Module 1: External Access Alerting

**Purpose:** Alert when any resource in the organization is shared publicly or with principals outside the org boundary.

**How it works:**
- Subscribes Access Analyzer findings to Security Hub
- EventBridge captures findings from ALL organization accounts (centralized via Security Hub aggregation)
- Lambda filters for HIGH and CRITICAL severity only
- Formatted Slack message with severity color-coding, account ID, region, resource name, and console link

**Terraform resources:**
- `aws_securityhub_product_subscription` — enables Access Analyzer → Security Hub integration
- `aws_cloudwatch_event_rule` — matches Security Hub findings with Access Analyzer product ARN
- `aws_lambda_function` — Python 3.11, formats and sends Slack notifications
- `aws_secretsmanager_secret` — stores Slack webhook URL (encrypted at rest)
- `aws_iam_role` — least-privilege (only `secretsmanager:GetSecretValue` on specific ARN)
- `aws_cloudwatch_log_group` — 14-day retention

**Key Lambda logic:**
```python
# Only HIGH and CRITICAL findings are sent to Slack
if severity not in ['HIGH', 'CRITICAL']:
    return {'statusCode': 200, 'body': 'Severity too low, skipped'}

# Structured Slack message with Block Kit
message = {
    "attachments": [{
        "color": severity_config[severity]['color'],
        "blocks": [header, title, fields, description, action_button]
    }]
}
```

## Module 2: Internal Access Alerting

**Purpose:** Detect cross-account resource sharing within the organization boundary.

**Example finding:** An S3 bucket in Account A has a policy granting access to Account B (both in the same org). This may be intentional but could also indicate overly broad sharing.

**How it works:**
- Deploys `ORGANIZATION_INTERNAL_ACCESS` analyzer with inclusion rules
- Only scans specified accounts + specified resource types (S3, RDS Snapshots, DynamoDB)
- EventBridge filters for CRITICAL severity only
- Lambda sends formatted Slack alert

**Terraform resources:**
- `aws_accessanalyzer_analyzer` — conditional (only if `enabled_accounts` is non-empty)
- `aws_cloudwatch_event_rule` — matches CRITICAL + Access Analyzer product + specific finding types
- `aws_lambda_function` — Python 3.11, CRITICAL-only processing
- `aws_iam_role` — references shared secret ARN from external access module

**EventBridge pattern (precise filtering):**
```json
{
  "source": ["aws.securityhub"],
  "detail-type": ["Security Hub Findings - Imported"],
  "detail": {
    "findings": {
      "ProductArn": [{"suffix": ":product/aws/access-analyzer"}],
      "Severity": {"Label": ["CRITICAL"]},
      "Types": [{"prefix": "Software and Configuration Checks/AWS Security Best Practices/Access Analyzer"}]
    }
  }
}
```

## Module 3: Unused Access Alerting

**Purpose:** Identify stale IAM permissions — roles not used, access keys never rotated, passwords unused.

**What it detects:**
- IAM roles with no activity in X days (configurable, default 90)
- Unused access keys
- Unused console passwords
- Unused service and action-level permissions

**How it works:**
- Deploys `ORGANIZATION_UNUSED_ACCESS` analyzer with inverted exclusion rules
- Dynamically computes exclusion list from org accounts minus opt-in list
- Configurable `unused_access_age` threshold
- EventBridge filters for CRITICAL + specific unused access finding types
- Lambda sends formatted Slack alert

**The exclusion inversion logic:**
```hcl
locals {
  all_account_ids = [for a in data.aws_organizations_organization.current[0].accounts : a.id]

  # AWS requires the analyzer owner account to always be scanned
  excluded_accounts = sort([
    for id in local.all_account_ids : id
    if !contains(var.enabled_accounts, id) && id != local.analyzer_account_id
  ])
}
```

**EventBridge pattern (unused access types):**
```json
{
  "detail": {
    "findings": {
      "ProductArn": [{"suffix": ":product/aws/access-analyzer"}],
      "Severity": {"Label": ["CRITICAL"]},
      "Types": [
        "Software and Configuration Checks/AWS Security Best Practices/Unused Permission",
        "Software and Configuration Checks/AWS Security Best Practices/Unused IAM Role",
        "Software and Configuration Checks/AWS Security Best Practices/Unused IAM User Password",
        "Software and Configuration Checks/AWS Security Best Practices/Unused IAM User Access Key"
      ]
    }
  }
}
```

## Slack Notification Examples

**External Access (HIGH):**
```
🟠 Access Analyzer Alert
━━━━━━━━━━━━━━━━━━━━━━━━
S3 bucket allows public read access

Severity:   🟠 HIGH
Account:    111111111111
Region:     eu-central-1
Resource:   my-public-bucket

[View in Access Analyzer]
```

**Internal Access (CRITICAL):**
```
🔴 Internal Access - CRITICAL Finding
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
S3 bucket grants cross-account access

Severity:   🔴 CRITICAL
Account:    222222222222
Region:     eu-central-1
Resource:   shared-data-bucket

[View in Access Analyzer]
```

**Unused Access (CRITICAL):**
```
🟠 Unused Access - CRITICAL Finding
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
IAM role admin-legacy-role has not been used in 120 days

Severity:   🔴 CRITICAL
Account:    111111111111
Region:     eu-central-1
Resource:   admin-legacy-role

[View in Access Analyzer]
```

## CI/CD Pipeline

- **Test workflow** (PRs): `terraform validate` + `terraform plan` — no infrastructure changes
- **Deploy workflow** (manual): `workflow_dispatch` with module dropdown selector
  - Targets a single module (e.g., `module.access_analyzer_unused`)
  - Or full deploy (all modules)
- **Authentication**: GitHub Actions OIDC — no long-lived credentials
- **State**: Remote S3 backend with DynamoDB locking

## Operational Considerations

### Analyzer Recreate Risk

Both paid analyzers (`internal` and `unused`) are **destroyed and recreated** when you add/remove accounts. This is an AWS API limitation.

**Impact:**
- All existing findings are permanently lost
- All accounts rescanned from scratch (double billing for that month)
- 24-hour blind spot until fresh scan completes

**Mitigation:**
- Batch account changes into a single deploy
- Deploy at start of month to minimize cost overlap
- `sort()` on exclusion lists prevents accidental recreates from list ordering drift

### Cost Model

| Component | Monthly Cost |
|-----------|-------------|
| External Access Analyzer | Free |
| Internal Access Analyzer | ~$0.20 per resource analyzed |
| Unused Access Analyzer | ~$0.20 per IAM entity analyzed |
| Lambda (3 functions) | ~$0.00 |
| Secrets Manager (1 shared secret) | $0.40 |
| CloudWatch Logs (3 log groups) | ~$0.30 |
| EventBridge (3 rules) | Free |
| **Total (2 accounts scanned)** | **~$1-5/month** |

Cost scales linearly with number of enabled accounts and resources/entities within them.

## Tech Stack

- **IaC**: Terraform (HCL) with AWS Provider >= 6.22
- **Runtime**: Python 3.11 (AWS Lambda)
- **AWS Services**: IAM Access Analyzer, Security Hub, EventBridge, Lambda, Secrets Manager, CloudWatch Logs, AWS Organizations
- **Notifications**: Slack Block Kit API
- **CI/CD**: GitHub Actions with OIDC authentication
- **State Management**: S3 + DynamoDB locking

## File Structure

```
modules/
├── access-analyzer-slack/          # External access + shared Slack secret
│   ├── main.tf                     # Provider + Security Hub subscription
│   ├── eventbridge.tf              # EventBridge rule (all org findings)
│   ├── lambda.tf                   # Lambda function + log group
│   ├── lambda_function.py          # Python handler (HIGH + CRITICAL)
│   ├── iam.tf                      # IAM role + policies
│   ├── secrets.tf                  # Secrets Manager (shared webhook)
│   ├── variables.tf                # Input variables
│   └── outputs.tf                  # Outputs (incl. secret_arn for sharing)
├── access-analyzer-internal/       # Internal/cross-account access
│   ├── main.tf                     # Analyzer (ORGANIZATION_INTERNAL_ACCESS, conditional)
│   ├── eventbridge.tf              # EventBridge rule (CRITICAL only)
│   ├── lambda.tf                   # Lambda function + log group
│   ├── lambda_function.py          # Python handler (CRITICAL only, cached webhook)
│   ├── iam.tf                      # IAM role + policies
│   ├── variables.tf                # Input variables
│   └── outputs.tf                  # Outputs
└── access-analyzer-unused/         # Unused IAM access detection
    ├── main.tf                     # Analyzer (ORGANIZATION_UNUSED_ACCESS, inverted exclusion)
    ├── eventbridge.tf              # EventBridge rule (CRITICAL + unused access types)
    ├── lambda.tf                   # Lambda function + log group
    ├── lambda_function.py          # Python handler (CRITICAL only, cached webhook)
    ├── iam.tf                      # IAM role + policies
    ├── variables.tf                # Input variables
    └── outputs.tf                  # Outputs
```

## Module Relationship

```
┌─────────────────────────────┐
│   access-analyzer-slack     │
│   (External Access - Free)  │
│                             │
│   Owns: Slack webhook secret│
│   Alerts: HIGH + CRITICAL   │
└──────────┬──────────────────┘
           │ secret_arn
           ├──────────────────────────────┐
           ▼                              ▼
┌─────────────────────────────┐  ┌─────────────────────────────┐
│  access-analyzer-internal   │  │  access-analyzer-unused     │
│  (Cross-Account - Paid)     │  │  (Stale IAM - Paid)         │
│                             │  │                             │
│  Uses: inclusion rules      │  │  Uses: inverted exclusion   │
│  Alerts: CRITICAL only      │  │  Alerts: CRITICAL only      │
│  Reuses: shared secret      │  │  Reuses: shared secret      │
└─────────────────────────────┘  └─────────────────────────────┘
```

## Author

**Muhammad Abdus Sobur**

Built as part of a security infrastructure initiative providing centralized IAM access monitoring and automated alerting for a multi-account AWS Organization.
