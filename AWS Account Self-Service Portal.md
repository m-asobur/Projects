# AWS Account Self-Service Portal

> A full-stack serverless web portal for provisioning and managing AWS accounts via Account Factory for Terraform (AFT). Users submit account requests through a React UI, which generates Terraform HCL files, creates Git branches, and opens Pull Requests automatically.

---

## Overview

This project automates the manual process of onboarding new AWS accounts in a multi-account AWS Organization. Instead of writing Terraform files by hand and raising PRs, users fill in a guided form, and the system handles code generation, branch creation, and PR submission — reducing time-to-provision from hours to minutes.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                  Identity Provider (Azure AD)                │
│                    (OIDC Federation)                         │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│         AWS Cognito User Pool (OAuth 2.0 PKCE)              │
│         Custom claims for RBAC (portal_role)                │
└────────────────────────┬────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│   CloudFront Distribution                                    │
│   ├── /* → S3 (React SPA)                                   │
│   └── /api/* → API Gateway → Lambda (Express)               │
└─────────────────────────────────────────────────────────────┘
                         │
                         ▼
┌─────────────────────────────────────────────────────────────┐
│   GitHub API                                                 │
│   Creates branches, commits .tf files, opens PRs             │
└─────────────────────────────────────────────────────────────┘
```

### Key Design Decisions

- **Serverless-first**: No VPCs, no EC2 — CloudFront + S3 + API Gateway + Lambda
- **GitOps pattern**: All infrastructure changes go through Git PRs (auditable, reviewable)
- **Monorepo**: npm workspaces with a shared package for types and validation schemas
- **External auth**: Leverages an existing central authentication system (Cognito + Azure AD federation)

---

## Tech Stack

| Layer       | Technology                                         |
|-------------|----------------------------------------------------|
| Frontend    | React 19, Vite 6, TypeScript, React Router 7       |
| Backend     | Express 4, TypeScript, Node.js 20                  |
| Auth        | Azure AD → AWS Cognito (OAuth 2.0 PKCE)           |
| Validation  | Zod (shared schemas between FE and BE)             |
| Forms       | React Hook Form                                    |
| Bundling    | esbuild (Lambda: ~336KB API, ~20KB scheduled job)  |
| Infra       | Terraform (S3, CloudFront, Lambda, API Gateway)    |
| Storage     | DynamoDB (request history, drafts with TTL)        |
| Git Client  | Octokit (GitHub REST API)                          |
| Testing     | Vitest, fast-check (property-based testing)        |
| State Mgmt  | S3 + DynamoDB (remote Terraform state + locking)   |

---

## Features

### Account Provisioning
- **New Workload Account** — guided form with auto-derived fields (email, account name, SSO credentials)
- **New Sandbox Account** — simplified form for developer sandboxes
- **Edit Existing Accounts** — reads current Terraform config from GitHub, presents editable form
- **Duplicate Detection** — checks if an account .tf file already exists before submission

### Developer Experience
- **Preview Panel** — shows generated HCL before submission
- **Client-side + Server-side Validation** — Zod schemas shared between frontend and backend
- **Field Derivation Engine** — computes 6+ fields automatically from minimal user input
- **Combo-box for Existing Projects** — reads folder structure from GitHub to offer autocomplete

### Workflow Automation
- **Automated PR Creation** — creates a Git branch, commits generated .tf files, opens a PR with labels
- **main.tf Module Registration** — auto-appends Terraform module block for new project folders
- **PR Status Sync** — scheduled Lambda polls GitHub every 5 min to update request statuses in DynamoDB

### Admin Capabilities
- **Review Dashboard** — admin-only view of all open requests
- **Edit & Push** — admins can modify submitted requests and push updated commits to existing PRs
- **Role-based Derived Field Editing** — admins can override auto-computed fields; viewers see read-only

### Security & Auth
- **OAuth 2.0 PKCE** — no client secrets in the browser
- **JWT Verification** — backend validates tokens against Cognito JWKS
- **RBAC via Custom Claims** — Pre Token Generation Lambda injects `portal_role` into JWTs
- **Session Timeout Warning** — 5-minute warning before token expiry
- **AuthGuard** — route protection on the frontend

---

## RBAC Model

| Role          | Permissions                                                  |
|---------------|--------------------------------------------------------------|
| super-admin   | Full access, override derived fields, review & edit PRs      |
| client-admin  | Same as super-admin, scoped to assigned application clients  |
| viewer        | Submit requests, view own history, read-only derived fields  |

---

## Project Structure

```
project-root/
├── apps/
│   ├── frontend/           # React + Vite SPA
│   │   ├── src/
│   │   │   ├── pages/      # 11 route-level pages (code-split)
│   │   │   ├── components/ # Reusable UI (Toast, Spinner, AuthGuard, etc.)
│   │   │   ├── contexts/   # AuthContext (token management, RBAC)
│   │   │   └── services/   # API client, auth service, mock API
│   │   └── index.html
│   └── backend/            # Express API (Lambda-compatible)
│       └── src/
│           ├── routes/     # accounts, config, review
│           ├── services/   # git-client, validation, derivation, hcl-generator
│           ├── middleware/ # auth, require-admin, error-handler
│           └── lambda.ts   # Lambda handler entry point
├── packages/
│   └── shared/             # Shared types, Zod schemas, constants
├── infra/                  # Terraform (unified single-apply)
│   ├── main.tf            # Provider, backend, common tags
│   ├── frontend.tf        # S3 + CloudFront + Route 53
│   ├── api.tf             # API Gateway + Lambda
│   ├── storage.tf         # DynamoDB tables
│   ├── iam.tf            # IAM roles + policies
│   └── pr-sync.tf        # Scheduled PR Status Sync Lambda
├── scripts/               # Deploy scripts (deploy.sh, bundle-lambda.mjs)
└── docs/                  # Architecture diagrams, ADRs, runbook
```

---

## Infrastructure (Terraform)

All infrastructure is managed as a single Terraform root module:

- **S3 Bucket** — static frontend hosting with versioning, encryption, public access block
- **CloudFront** — Origin Access Control for S3, API Gateway as second origin for `/api/*`
- **API Gateway** — HTTP API with CORS, auto-deploy stage, structured access logging
- **Lambda** — Node.js 20 runtime, Express app wrapped with serverless adapter
- **DynamoDB** — two tables: request history (userId + createdAt) and drafts (with TTL)
- **Route 53** — custom domain A record aliased to CloudFront
- **IAM** — least-privilege roles for Lambda (DynamoDB, CloudWatch, SSM read)
- **Remote State** — S3 backend with DynamoDB locking

---

## Backend Design

### Request Flow

```
User submits form
    → Frontend validates (Zod schema)
    → POST /api/accounts/workload
    → Server-side validation (duplicate check, field format)
    → Derive computed fields (email, account name, SSO user)
    → Generate HCL (Terraform .tf file content)
    → Create Git branch (from main)
    → Commit .tf file to branch
    → Register module in main.tf (if new project)
    → Open Pull Request with labels
    → Save to DynamoDB (request history)
    → Return PR URL to frontend
```

### Services

| Service         | Responsibility                                                   |
|-----------------|------------------------------------------------------------------|
| GitClient       | GitHub API wrapper (branches, commits, PRs, file reads)          |
| Derivation      | Computes email, account name, SSO fields, file paths from input  |
| HCL Generator   | Produces Terraform-compatible .tf file content                   |
| Validation      | Zod-based + business rules (duplicate detection, immutable fields)|
| DynamoDB History | Persistence layer for request tracking                           |

### API Endpoints

| Method | Path                              | Description                    | Auth     |
|--------|-----------------------------------|--------------------------------|----------|
| GET    | /api/health                       | Health check                   | None     |
| POST   | /api/accounts/workload            | Create workload account PR     | Auth     |
| POST   | /api/accounts/sandbox             | Create sandbox account PR      | Auth     |
| PUT    | /api/accounts/workload/:app/:env  | Update existing workload       | Auth     |
| PUT    | /api/accounts/sandbox/:name       | Update existing sandbox        | Auth     |
| GET    | /api/accounts/history             | User's request history         | Auth     |
| GET    | /api/review/requests              | All open requests (admin)      | Admin    |
| PUT    | /api/review/edit-pr               | Edit and push to existing PR   | Admin    |
| GET    | /api/config/projects              | List existing project folders  | Auth     |

---

## Frontend Design

### Key Patterns

- **Code Splitting** — every page is lazy-loaded via `React.lazy()`
- **Error Boundary** — graceful error recovery with retry
- **Toast Notifications** — context-based toast system
- **Auth Context** — manages tokens, groups, `isAdmin` derivation, session warnings
- **Form Validation** — Zod schemas used both client-side and server-side (shared package)

### Pages

| Page                    | Purpose                                      |
|-------------------------|----------------------------------------------|
| Home                    | Dashboard with navigation cards              |
| New Workload Account    | Multi-field form with derivation + preview   |
| New Sandbox Account     | Simplified sandbox request form              |
| Edit Account Selector   | Choose workload/sandbox + project to edit    |
| Edit Workload Account   | Pre-filled form from existing GitHub .tf     |
| Edit Sandbox Account    | Pre-filled sandbox edit form                 |
| Request History         | User's submitted PRs with status tracking    |
| Review Requests         | Admin view of all open requests              |
| Callback               | OAuth 2.0 token exchange handler             |
| Logged Out             | Post-logout landing page                     |

---

## Deployment

- **esbuild** bundles TypeScript into single .mjs files for Lambda (~336KB)
- **Scripts**: `deploy.sh` (full), `deploy-lambda.sh` (backend-only, ~5 seconds)
- **CloudFront Invalidation** on every deploy
- **GitHub Actions** — CI/CD with OIDC-based AWS authentication (no long-lived keys)

---

## What I Built & My Role

- Designed and implemented the full-stack architecture from scratch
- Built the monorepo structure with shared validation schemas
- Implemented the GitOps workflow (branch → commit → PR automation via Octokit)
- Designed the RBAC system with custom Cognito claims
- Created the Terraform infrastructure (serverless, single-apply)
- Implemented property-based testing for business logic (derivation, HCL generation)
- Set up deployment scripts with esbuild bundling

---

## Lessons Learned

- **Shared Zod schemas** eliminate validation drift between frontend and backend
- **esbuild** reduces Lambda cold starts significantly vs. webpack
- **GitOps for infrastructure** provides full auditability via PR history
- **DynamoDB on-demand** is cost-effective for low-traffic internal tools
- **CloudFront as unified entry point** simplifies CORS and certificate management

---

*This document describes a personal portfolio project. All organization-specific identifiers, domains, account IDs, and credentials have been removed.*
