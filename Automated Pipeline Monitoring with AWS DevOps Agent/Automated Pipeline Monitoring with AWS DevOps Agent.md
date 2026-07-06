# Automated Pipeline Monitoring with AWS DevOps Agent

## Project Overview

Designed and implemented an automated incident response system that detects pipeline failures and delivers AI-powered root cause analysis to Slack within minutes — eliminating manual log investigation.

## The Problem

A video-to-article processing system runs multiple Step Function pipelines. When failures occurred, the team wouldn't know until someone manually checked logs — often hours later. Debugging required correlating CloudWatch logs, metrics, and execution history across multiple services.

## The Solution

Built an event-driven pipeline that:

1. **Detects** failures automatically via EventBridge
2. **Triggers** a Lambda function that constructs and HMAC-signs a webhook payload
3. **Sends** the payload to AWS DevOps Agent (a managed AI service)
4. **Investigates** autonomously — correlating logs, metrics, and deployment data
5. **Delivers** root cause analysis + mitigation steps to Slack

## Architecture

```
Step Function fails
      ↓
EventBridge Rule (matches FAILED/TIMED_OUT/ABORTED)
      ↓
Webhook Caller Lambda
  ├── Reads HMAC secret from Secrets Manager
  ├── Constructs incident payload
  ├── Signs with HMAC-SHA256
  └── POSTs to DevOps Agent webhook
      ↓
AWS DevOps Agent (autonomous investigation)
  ├── Reads CloudWatch logs
  ├── Analyzes metrics
  └── Correlates execution history
      ↓
Slack (root cause + mitigation recommendations)
```

## Key Metrics

| Before | After |
|--------|-------|
| Detection: hours (manual) | Detection: < 5 minutes (automatic) |
| Investigation: 15-30 min manual log diving | Investigation: zero manual effort |
| Coverage: whoever remembers to check | Coverage: 100% of pipeline failures |

## Technologies Used

- **AWS Lambda** (Python 3.11) — Webhook caller function
- **Amazon EventBridge** — Event-driven failure detection
- **AWS Secrets Manager** — Secure HMAC key storage
- **AWS DevOps Agent** — Managed AI incident investigation
- **HMAC-SHA256** — Cryptographic payload signing
- **Amazon SQS** — Dead-letter queue for reliability
- **Amazon CloudWatch** — Monitoring and alerting

## What I Built

### Lambda Function (7 modules)

| Module | Purpose |
|--------|---------|
| `handler.py` | Orchestrates the full flow |
| `payload.py` | Transforms events into webhook schema |
| `signing.py` | HMAC-SHA256 signing with replay protection |
| `secrets.py` | Secrets Manager client with error handling |
| `delivery.py` | HTTP POST with exponential backoff retry |
| `models.py` | Constants and priority mapping |

### Infrastructure

- EventBridge rule matching 5 state machines for 3 failure types
- IAM role with least-privilege (only Secrets Manager read + CloudWatch Logs)
- SQS dead-letter queue + CloudWatch alarm for system reliability
- Complete setup documentation for the team

### Security Design

- HMAC-SHA256 payload signing (integrity + replay protection)
- Secrets stored in Secrets Manager (not env vars)
- Least-privilege IAM — Lambda can only read one specific secret
- No sensitive data in code

## How HMAC Signing Works

```
timestamp = current time (ISO 8601)
payload = JSON.dumps(incident_data, compact=True)
message = f"{timestamp}:{payload}"
signature = Base64(HMAC-SHA256(secret_key, message))

Headers:
  x-amzn-event-timestamp: <timestamp>
  x-amzn-event-signature: <signature>
  Content-Type: application/json
```

This provides:
- **Integrity** — any payload modification invalidates the signature
- **Authentication** — only the holder of the secret can produce valid signatures
- **Replay protection** — timestamp included in signature, stale requests rejected

## Code Sample: Payload Construction

```python
def build_webhook_payload(event: dict) -> dict:
    detail = event["detail"]
    execution_arn = detail["executionArn"]
    status = detail["status"]

    # Derive state machine name from ARN
    sm_name = STATE_MACHINE_NAME_REGEX.match(detail["stateMachineArn"]).group(1)

    # Map status to priority
    priority = {"FAILED": "CRITICAL", "TIMED_OUT": "HIGH", "ABORTED": "MEDIUM"}[status]

    # Deterministic incident ID from execution ARN
    incident_id = hashlib.sha256(execution_arn.encode()).hexdigest()

    return {
        "eventType": "incident",
        "incidentId": incident_id,
        "action": "created",
        "priority": priority,
        "title": f"[{sm_name}] EXECUTION_{status}",
        "description": f"Execution {execution_arn} failed...",
        "timestamp": datetime.now(tz=timezone.utc).isoformat(),
        "service": sm_name,
        "data": detail,
    }
```

## Code Sample: HTTP Delivery with Retry

```python
def deliver_webhook(url: str, payload: dict, headers: dict) -> bool:
    """Delivers with exponential backoff: 1s, 2s between retries."""
    for attempt in range(3):
        try:
            request = urllib.request.Request(url, data=body, headers=headers, method="POST")
            response = urllib.request.urlopen(request)
            if 200 <= response.getcode() < 300:
                return True
        except urllib.error.HTTPError:
            if attempt < 2:
                time.sleep([1, 2][attempt])
    return False
```

## My Role

- Designed the architecture (chose webhook approach over direct API call for decoupling)
- Implemented all Lambda code (7 Python modules)
- Set up the full AWS infrastructure (Lambda, EventBridge, IAM, Secrets Manager, SQS, CloudWatch)
- Configured the DevOps Agent (Agent Space, webhook, Slack integration, investigation Skill)
- Wrote comprehensive setup documentation for the team
- Deployed and verified in production
- Created teardown documentation for clean removal

## Outcome

System deployed to production. When pipelines fail, the team receives automated root cause analysis in Slack within minutes — no manual investigation needed. Runs alongside the existing alerting system for both immediate notification AND deep analysis.
