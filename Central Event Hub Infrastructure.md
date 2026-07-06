# Central Event Hub Infrastructure

> A production-grade, multi-environment event-driven architecture built with AWS CDK (TypeScript), featuring EventBridge, SNS, schema validation, cross-account access management, and automated CI/CD pipelines.

## 🏗️ Architecture

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Publishers    │───▶│  EventBridge     │───▶│   SNS Topics    │───▶│   Subscribers   │
│ (Applications)  │    │  Custom Bus      │    │ (Routing)       │    │ (Applications)  │
└─────────────────┘    └──────────────────┘    └─────────────────┘    └─────────────────┘
                                │
                                ▼
                       ┌─────────────────┐
                       │ Schema Registry │
                       │ (Validation)    │
                       └─────────────────┘
```

### Event Flow
1. **Applications publish events** to an EventBridge custom bus (ingress)
2. **Lambda validator** validates event structure against JSON schemas
3. **EventBridge rules** route events based on source/detail-type patterns
4. **SNS topics** deliver events to subscribers (same-account and cross-account)
5. **CloudWatch** dashboards and alarms monitor all components
6. **Dead Letter Queues** capture failed validations with detailed error messages

## 📁 Project Structure

```
central-event-hub-infrastructure/
├── bin/                           # CDK app entry point
│   └── central-event-hub.ts       # Multi-environment setup
├── lib/                           # Stack definitions
│   ├── central-event-hub-stack.ts # Main infrastructure stack
│   └── constructs/                # Reusable CDK constructs
│       ├── event-router.ts        # Event routing logic
│       ├── schema-registry.ts     # Schema management
│       ├── monitoring-dashboard.ts # CloudWatch monitoring
│       └── cross-account-sns.ts   # Cross-account SNS access
├── lambda/                        # Lambda functions
│   ├── event-validator/           # Schema validation Lambda
│   └── reference/                 # Reference producer & consumer
├── test/                          # Unit tests (14 tests, 100% coverage)
├── packages/                      # NPM packages
│   └── event-types/               # Auto-generated TypeScript types
├── scripts/                       # Utility scripts
│   ├── generate-types.ts          # Type generation from schemas
│   └── schema-cli.ts             # Schema management CLI
├── contracts/                     # Event contracts (JSON)
├── .github/workflows/             # CI/CD pipeline (6 workflows)
└── docs/                          # Comprehensive documentation
```

## 🚀 Key Features

### Event-Driven Architecture
- **EventBridge Custom Bus** for centralized event ingestion
- **Pattern-based routing** to domain-specific SNS topics
- **Multi-target delivery** — events route to domain + aggregate topics
- **Schema validation** via Lambda before routing (fail-fast with DLQ)

### Multi-Environment Infrastructure
- **4 environments**: dev, int, staging, production
- **Separate AWS accounts** per environment for isolation
- **Branch-based deployments**: each branch maps to an environment
- **Manual approval gates** for production deployments

### Schema Management System
- **Centralized Schema Registry** using AWS EventBridge Schema Registry
- **CLI tools** for schema operations: add, validate, list, generate types
- **JSON Schema Draft 4** validation with detailed error messages
- **Schema evolution policies** with backward compatibility enforcement
- **Versioning** support for all schemas

### TypeScript Type Auto-Generation
- **Automated pipeline** generates TypeScript types from Schema Registry
- **Published as NPM package** for type-safe event publishing/consuming
- **Semantic versioning** with automatic patch bumps on deployments
- **GitHub Actions integration** triggers generation after successful deploys

### Cross-Account Access Management
- **Resource-based SNS policies** for cross-account subscriptions
- **DynamoDB-driven access control** — register consumers in a table, policies auto-update
- **Lambda policy updater** triggered by DynamoDB streams
- **Least privilege**: Subscribe/Receive only, no Publish or Delete
- **Monitoring dashboard** for cross-account access visibility

### CI/CD Pipeline (GitHub Actions)
- **OIDC authentication** — no stored AWS credentials
- **6 workflows**: deploy, reusable deploy, rollback, delete, drift detection, type generation
- **Multi-stage pipeline**: Tests → Build → Synth → Deploy → Monitor
- **Emergency rollback** capability via manual workflow dispatch
- **Infrastructure drift detection** on schedule

### Observability & Monitoring
- **CloudWatch Dashboards** per environment (EventBridge + SNS metrics)
- **CloudWatch Alarms** on all SNS topics for delivery failures
- **Dead Letter Queues** for failed event validations
- **Detailed logging** in all Lambda functions

## 🛠️ Technology Stack

| Layer | Technology |
|-------|-----------|
| **IaC** | AWS CDK (TypeScript) |
| **Compute** | AWS Lambda (Node.js) |
| **Event Bus** | Amazon EventBridge |
| **Messaging** | Amazon SNS |
| **Queuing** | Amazon SQS (DLQ) |
| **Schema** | EventBridge Schema Registry |
| **Storage** | Amazon DynamoDB |
| **Monitoring** | Amazon CloudWatch |
| **CI/CD** | GitHub Actions |
| **Auth** | AWS IAM + OIDC Federation |
| **Testing** | Jest |

## 📊 Infrastructure Resources (Per Environment)

- 1× EventBridge Custom Bus (ingress)
- 5× SNS Topics (domain-specific + aggregate)
- 2× EventBridge Routing Rules
- 1× Schema Registry with 2+ schemas
- 1× CloudWatch Dashboard
- 5× CloudWatch Alarms
- 1× Event Validator Lambda
- 1× DynamoDB Table (access control)
- 1× Policy Updater Lambda (DynamoDB Stream)
- 1× Validation DLQ (SQS)

## 🔄 Event Routing Examples

### Domain Event Routing
```json
// Events from payment service route to Payment Topic + User Events Topic
{
  "source": ["pub::payment"],
  "detail-type": ["Payment Transaction", "Payment Status"]
}
```

```json
// Events from DCC service route to DCC Topic + User Events Topic
{
  "source": ["pub::dcc"]
}
```

### Standard Event Envelope
```typescript
interface Event {
  id: string;              // UUID
  source: string;          // pub::{service-name}
  'detail-type': string;   // Event type
  time: string;            // ISO 8601 timestamp
  detail: object;          // Event-specific payload
}
```

### Publishing Events (SDK)
```typescript
import { EventBridgeClient, PutEventsCommand } from '@aws-sdk/client-eventbridge';

const client = new EventBridgeClient({ region: 'eu-central-1' });

await client.send(new PutEventsCommand({
  Entries: [{
    Source: 'pub::order-service',
    DetailType: 'Order Created',
    Detail: JSON.stringify({
      orderId: 'ord-123',
      customerId: 'cust-456',
      amount: 99.99,
      currency: 'EUR'
    }),
    EventBusName: 'central-event-hub-dev'
  }]
}));
```

## 🔐 Security Design

- **No stored credentials** — OIDC federated authentication for CI/CD
- **Environment isolation** — separate AWS accounts per stage
- **Least privilege** — resource-based policies grant minimal permissions
- **Manual approval** — production deployments require reviewer sign-off
- **Audit trail** — all deployments logged in GitHub Actions
- **DynamoDB access control** — consumer accounts must be explicitly registered

## 🧪 Testing

- **14 unit tests** covering all constructs and stack configurations
- **100% test coverage** for infrastructure code
- **Reference producer/consumer Lambdas** for end-to-end testing
- **Schema validation testing** (valid events pass, invalid rejected to DLQ)
- **Cross-account testing** procedures documented

```bash
yarn test           # Run all unit tests
yarn test --coverage # Run with coverage report
yarn build          # Compile TypeScript
yarn cdk synth      # Synthesize CloudFormation templates
yarn cdk diff       # Preview infrastructure changes
```

## 📦 CDK Constructs (Reusable)

### EventRouter
Manages EventBridge rules and multi-target routing logic.

```typescript
router.addRule('OrderEventsRule', {
  ruleName: `order-routing-${environment}`,
  eventPattern: { source: ['pub::order-service'] },
  targets: [topics.orderEvents, topics.allEvents]
});
```

### SchemaRegistry
Centralized schema registration with JSON Schema validation.

```typescript
schemaRegistry.addSchema('OrderCreatedEvent', {
  description: 'Event emitted when order is created',
  schema: { /* JSON Schema */ }
});
```

### CrossAccountSNS
Resource-based policy management for cross-account SNS access.

```typescript
new CrossAccountSNS(this, 'CrossAccountAccess', {
  topics: snsTopics,
  targetAccountIds: ['123456789012']
});
```

### MonitoringDashboard
CloudWatch dashboard and alarm configuration.

```typescript
new MonitoringDashboard(this, 'Monitoring', {
  topics: snsTopics,
  environment: 'dev'
});
```

## 📚 Documentation Suite

| Document | Purpose |
|----------|---------|
| API Reference | Complete construct and CLI API docs |
| Deployment Guide | Multi-environment deployment procedures |
| Schema Management | Schema registry and CLI tools guide |
| Schema Best Practices | Event schema design standards |
| Event Registration Tutorial | Step-by-step guide for new events |
| Cross-Account SNS | Cross-account subscription setup |
| Testing Guide | Producer/consumer testing procedures |
| Troubleshooting | Common issues and solutions |
| Contributing Guide | Development workflow and standards |
| Environment Setup | GitHub environment configuration |
| AWS OIDC Setup | Authentication configuration |

## 🎯 Project Objectives (All Complete)

| # | Objective | Description |
|---|-----------|-------------|
| 1 | AWS Accounts Setup | Multi-account strategy with dedicated accounts per environment |
| 2 | Repository & CDK Setup | TypeScript CDK project with comprehensive testing |
| 3 | CI/CD Pipeline | Branch-based automated deployments with OIDC auth |
| 4 | EventBridge Infrastructure | Event routing with SNS topics and monitoring |
| 5 | Schema Management | Registry, validation CLI, and evolution policies |
| 6 | TypeScript Library | Auto-generated types from schemas as NPM package |
| 7 | Cross-Account Access | Resource-based policies with DynamoDB access control |
| 8 | Documentation | Complete docs suite with tutorials and best practices |

## 🏃 Quick Start

```bash
# Install dependencies
yarn install

# Build the project
yarn build

# Run tests
yarn test

# Schema operations
yarn schema:list                    # List all schemas
yarn schema:validate --name MyEvent # Validate a schema
yarn schema:add --name NewEvent --file ./path/to/schema.ts
yarn schema:generate-types          # Generate TypeScript types

# CDK operations
yarn cdk synth     # Generate CloudFormation
yarn cdk diff      # Preview changes
yarn cdk deploy    # Deploy infrastructure
```

## 📈 Highlights

- **Production-ready**: Running across 4 environments with real event traffic
- **Fully automated**: Zero-touch deployments from code push to infrastructure update
- **Type-safe**: Auto-generated TypeScript types ensure compile-time event safety
- **Extensible**: Adding new events requires only a schema file + routing rule
- **Observable**: Full monitoring with dashboards, alarms, and DLQ visibility
- **Secure**: OIDC auth, account isolation, least privilege, manual prod approval
- **Well-documented**: 11 documentation files covering all aspects of the system
