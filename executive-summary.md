# Platform Executive Summary

**Document purpose**: Non-technical overview of the platform -- what it does, how it works, and why we built it this way. This document is domain-agnostic and can be adapted for any SaaS project built on this stack.

**Audience**: Product owners, operations leads, and business leadership. No programming knowledge required.

---

## 1. What We Built

We built a cloud-based SaaS platform composed of two applications that work together as a unified system.

### The Two Applications

| Application | Users | Device | Purpose |
|-------------|-------|--------|---------|
| **Admin Portal** | Administrators, reviewers, analysts | Desktop or laptop | Manage data, review records, configure rules, run reports |
| **Mobile App** (Progressive Web App) | Field workers, inspectors, operators | Mobile phone | Capture data on-site, photograph items, complete guided workflows |

Both applications connect to the same shared cloud backend. Data entered on a phone in the field is immediately visible to a reviewer on a desktop in an office. There is no delay, no batch sync, and no manual export/import step.

### Why Two Apps Instead of One?

Desktop users and field workers have fundamentally different needs. The Admin Portal is optimized for large screens, data tables, and complex filtering. The Mobile App is optimized for one-handed use, camera access, and working in environments with intermittent connectivity. Building two purpose-built interfaces on a shared backend gives each user group the best possible experience without duplicating data or logic.

### Shared Backend

Both applications share a single backend powered by Supabase, an open-source platform that bundles a database, user authentication, file storage, serverless functions, and real-time data sync into one managed service. This means there is exactly one source of truth for all data, and both apps always see the same information.

---

## 2. Core Capabilities

### Entity Lifecycle Tracking

The platform tracks entities (the core business objects) through a defined lifecycle. Each entity moves through stages -- for example, created, in progress, under review, and completed. At each stage, the system captures relevant data, photos, and status changes. The complete history is always available.

### Photo-Based Workflows with AI Analysis

The Mobile App guides field workers through structured steps using their phone camera. Photos are uploaded automatically and linked to the relevant record. Once uploaded, photos are analyzed by AI in the background -- no manual review step required. Results appear in real time on both apps.

### Real-Time Updates Across All Users

When data changes anywhere in the system -- a new record is created, a photo is analyzed, a status is updated -- every connected user sees the change immediately. There is no polling, no refresh button, and no stale data. This is powered by WebSocket connections (persistent two-way channels between the browser and the server).

### Role-Based Access Control

Users are assigned roles that determine what they can see and do. A field worker sees only the records assigned to them. A reviewer sees all records awaiting review. An administrator can configure system rules and manage users. These permissions are enforced at the database level, not just in the user interface, so they cannot be bypassed.

### Multi-Tenant Architecture

The platform supports multiple independent organizations (tenants) in a single deployment. Each tenant's data is completely isolated from every other tenant's data. This isolation is enforced by Row-Level Security (RLS) -- a database feature that automatically filters every query to include only the current tenant's data. Even if application code had a bug, the database would still prevent cross-tenant data leakage.

The platform can also be deployed in single-tenant mode for organizations that require a dedicated instance.

### Offline-Capable Mobile Experience

The Mobile App is a Progressive Web App (PWA), which means it can be installed on a phone like a native app and continue to function with limited or no connectivity. Field workers can capture data and photos while offline; the app synchronizes automatically when connectivity is restored.

### Internationalization

The platform ships with full support for English and Spanish. Every piece of user-facing text -- labels, messages, error descriptions, help content -- is stored in translation files rather than embedded in code. Adding a new language requires translating these files; no code changes are needed.

---

## 3. AI Capabilities

### Automatic Image Analysis

When a field worker uploads a photo through the Mobile App, the system automatically:

1. Saves the photo securely in cloud storage
2. Queues an analysis job (no human action required)
3. Sends the photo to an AI vision model with specific detection instructions
4. Receives a structured response (classification, tags, confidence score)
5. Writes the result to the database and notifies all connected users in real time

This process typically completes in a few seconds and happens entirely in the background. Field workers do not wait for results; they continue their workflow while the AI processes each photo.

### Configurable Detection Rules

The AI does not have hardcoded rules. Detection behavior is controlled by two types of configurable records in the database:

- **Tags**: Define what to look for in an image (e.g., specific conditions, objects, or states)
- **Rules**: Define business logic that maps combinations of tags to classifications (e.g., "if any of these conditions are present, classify as non-compliant")

Operations staff can add, modify, or disable tags and rules through the Admin Portal without a software deployment. This means the AI's behavior can be tuned by the business team, not just by developers.

### Real-Time Result Delivery

When the AI finishes analyzing a photo, the result is broadcast to all connected users via real-time channels. The Admin Portal updates dashboards and record views instantly. The Mobile App confirms the submission status to the field worker. No one has to refresh a page or check back later.

### Complete Audit Trail

Every AI analysis is logged with full detail:

- Which AI model was used (and which version)
- How many tokens were consumed (directly tied to cost)
- How long the analysis took
- The complete input and output
- The confidence score and final classification

This audit trail supports compliance requirements, cost tracking, and debugging. If a result is ever questioned, the complete record of how the AI arrived at its conclusion is available.

### Eval and Regression Framework

Changing AI detection rules carries risk -- a well-intentioned tweak could cause the AI to miss important conditions or generate false alarms. To prevent this, the platform includes a built-in evaluation system:

1. The team maintains a library of test cases -- photos with known correct answers (the "golden set")
2. Before any rule change is promoted to production, the system runs the AI against all test cases using the proposed new rules
3. Results are scored for precision (did it avoid false alarms?) and recall (did it catch all real issues?)
4. If scores drop below acceptable thresholds, the change is rejected or revised
5. A complete history of all configuration versions is retained for comparison and rollback

This means the team can tune AI behavior with confidence, knowing any degradation will be caught before it reaches production.

### Dead Letter Queue -- No Photo Left Behind

Photos that fail AI analysis (due to network errors, unexpected content, or transient failures) are not silently dropped. They are moved to a dead letter queue (DLQ) -- a holding area for failed items that need investigation. The DLQ is a standard database table that administrators can review, diagnose, and manually reprocess. The system guarantees that every uploaded photo is either successfully analyzed or explicitly flagged for human attention.

---

## 4. Technology Choices

Every technology was chosen to minimize operational complexity, maximize developer productivity, and keep costs low for small teams.

| Layer | Technology | Why We Chose It |
|-------|-----------|-----------------|
| Frontend framework | React + Vite | Fast build times (sub-second hot reload during development), massive ecosystem of libraries, largest hiring pool of any frontend framework |
| Backend platform | Supabase | Bundles database, authentication, file storage, serverless functions, and real-time sync into one managed service -- eliminates five separate infrastructure decisions |
| AI provider | OpenAI Vision API | Industry-leading image analysis accuracy, well-documented API, predictable per-image pricing |
| Frontend hosting | Vercel | Zero-configuration deployment, automatic preview environments for every pull request, global CDN for fast page loads worldwide |
| CI/CD pipeline | GitHub Actions | Native integration with our code repository, free for open-source projects, extensive marketplace of pre-built automation steps |
| Monorepo tooling | Nx | Detects which parts of the codebase changed and only runs tests/builds for affected projects -- keeps CI fast as the codebase grows |
| Observability | New Relic | Free tier includes 100 GB per month of data ingestion and one full-access user -- sufficient for a startup; unified platform for logs, metrics, and traces |
| End-to-end testing | Playwright | Tests both apps in real browsers (Chrome, Firefox, Safari), simulates complete user workflows including photo capture |
| Unit testing | Vitest | Fast, modern test runner that shares configuration with our build tool (Vite), reducing setup overhead |

### Why These Choices Matter for the Business

- **Managed services over self-hosted**: Supabase, Vercel, and GitHub handle infrastructure, security patches, and scaling. The team focuses on product features, not server maintenance.
- **Open-source foundations**: React, Vite, Supabase, and Playwright are all open-source. There is no vendor lock-in on the core technologies. If any managed service becomes unsuitable, the underlying technology can be self-hosted.
- **Pay-as-you-grow pricing**: Every service offers a free or low-cost tier suitable for development and early production. Costs scale with usage, not with upfront commitments.

---

## 5. Integration Overview

The platform is built on three cloud services that integrate with each other automatically.

### How the Three Platforms Connect

```
GitHub                    Supabase                   Vercel
(code + automation)       (backend + database)       (frontend hosting)
        |                        |                        |
        |--- PR opened -------->|--- creates preview DB   |
        |                        |   (isolated copy)       |
        |--- PR opened ---------------------------------->|--- deploys preview apps
        |                        |                        |   (connected to preview DB)
        |                                                 |
        |--- PR merged -------->|--- applies DB changes   |
        |                        |   to production         |
        |--- PR merged ---------------------------------->|--- deploys to production
        |                        |                        |   (global CDN)
        |                                                 |
        |--- PR closed -------->|--- deletes preview DB   |
        |--- PR closed ---------------------------------->|--- removes preview apps
```

### GitHub -- Code and Automation

GitHub is the central hub. All code lives in a single repository. Every change goes through a pull request (a formal proposal to merge new code). GitHub Actions -- an automation engine built into GitHub -- runs tests, checks code quality, and orchestrates deployments.

### Supabase -- Backend and Database

Supabase provides the database (PostgreSQL), user authentication, file storage for photos and documents, serverless functions for AI processing, and real-time data channels. Supabase's GitHub integration automatically creates an isolated database copy for each pull request so that new features can be tested against real data without affecting production.

### Vercel -- Frontend Hosting

Vercel hosts both web applications. Each app is a bundle of files deployed to a global content delivery network (CDN) -- a network of servers distributed worldwide that serves files from the location nearest to the user. Vercel's GitHub integration automatically deploys a preview version of each app for every pull request.

### The Key Insight

All three platforms watch the same GitHub repository and react to the same events (pull request opened, code merged, pull request closed). This means the entire workflow -- from code change to deployed preview to production release -- is fully automated with no manual steps.

---

## 6. Preview Environments

Every pull request gets a complete, isolated copy of the entire platform.

### What Gets Created

When a developer opens a pull request, three things happen automatically:

1. **Supabase** creates an isolated database with all schema changes applied and test data loaded
2. **Vercel** deploys preview versions of both the Admin Portal and the Mobile App
3. **A bot posts a comment** on the pull request with links to everything

The preview apps are connected to the preview database. They function exactly like the production system but with test data and the proposed code changes.

### What This Looks Like

```
Pull Request #42: "Add new reporting dashboard"

  Preview Database:   Isolated copy with test data
  Admin Portal:       https://portal-pr-42.example.app
  Mobile App:         https://mobile-pr-42.example.app
```

### Why This Matters

- **Stakeholders can review features** before they reach production by clicking a link in the pull request
- **QA can test changes** against realistic data without affecting the live system
- **Developers can demo work** to product owners without setting up a local environment
- **No shared staging environment** that accumulates configuration drift or test data corruption

### Automatic Cleanup

When a pull request is closed (merged or abandoned), all preview resources are deleted automatically. The database copy is removed. The preview apps are taken down. There are no orphaned environments accumulating cost or confusion.

---

## 7. Quality Assurance

### What Runs on Every Pull Request

When a developer opens or updates a pull request, the following checks run automatically in parallel:

| Check | What It Does | Typical Duration |
|-------|-------------|-----------------|
| **Linting** | Scans code for style violations and common errors | ~30 seconds |
| **Unit tests** | Runs fast, isolated tests on individual functions | ~1 minute |
| **Coverage report** | Measures what percentage of code is exercised by tests | Posted as a PR comment |
| **Preview deployments** | Builds and deploys live previews of both apps | ~2 minutes |

A typical pull request receives all results within 3 to 5 minutes.

### End-to-End Test Coverage

The platform includes over 100 automated end-to-end (E2E) tests that exercise both applications in real browsers. These tests simulate actual user workflows:

- **Admin Portal tests** verify that data displays correctly in tables and grids, that filtering and sorting work, and that configuration changes take effect. Portal tests are read-only -- they verify pre-existing data rather than creating new records.
- **Mobile App tests** simulate complete field workflows including form filling, photo capture, and multi-step processes. Each test creates its own data, runs the workflow, and verifies the result.

E2E tests run against a full local stack (database, backend services, and both applications) to ensure that every layer of the system works together correctly.

### What Runs on Merge to Production

When code is merged to the main branch, a stricter pipeline runs:

1. All lint checks and tests run across the entire codebase (not just affected projects)
2. Database schema changes are applied to the production database
3. Serverless functions are deployed to production
4. The Admin Portal is deployed to production (globally, via CDN)
5. The Mobile App is deployed to production (globally, via CDN)

The deployment order is enforced: **database changes land before frontend code**. This prevents the new frontend from running queries against a schema that does not yet have the expected tables or columns.

### How Rollback Works

Every production deployment corresponds to a specific version of the code. Rolling back a problematic deployment means creating a pull request that reverts the bad change. The same automated pipeline runs, and the previous working version is live within minutes. There is no special rollback procedure or emergency access required.

---

## 8. Deployment Flow

### The Complete Path from Code to Production

```
Developer writes code on a feature branch
    |
    v
Developer opens a pull request on GitHub
    |
    +---> GitHub Actions: lint code, run unit tests
    |
    +---> Supabase: create isolated preview database
    |         apply all database schema changes
    |         load test data
    |
    +---> Vercel: deploy preview Admin Portal
    |              deploy preview Mobile App
    |              connect previews to preview database
    |
    v
Team reviews code, tests preview, approves pull request
    |
    v
Developer merges pull request to main branch
    |
    v
GitHub Actions runs the production pipeline (in strict order):
    |
    |  Step 1: Validate
    |    - Lint entire codebase
    |    - Run all unit tests
    |
    |  Step 2: Deploy backend (must complete before Step 3)
    |    - Apply database migrations to production
    |    - Deploy serverless functions to production
    |
    |  Step 3: Deploy frontends (runs after Step 2 succeeds)
    |    - Deploy Admin Portal to Vercel (global CDN)
    |    - Deploy Mobile App to Vercel (global CDN)
    |
    v
Production is updated. Zero downtime.
    |
    v
Pull request closed: all preview environments deleted automatically
```

### Key Properties

- **Database-first deployment**: Schema changes are always applied before the frontend that depends on them. This prevents errors caused by the app expecting columns or tables that do not yet exist.
- **Zero-downtime deployments**: Vercel performs atomic deployments -- the old version serves traffic until the new version is fully ready, then traffic switches instantly.
- **Automatic cleanup**: Preview databases and preview apps are deleted when the pull request is closed, whether it was merged or abandoned.
- **No manual steps**: The entire flow from merge to production is automated. No one needs to SSH into a server, run a deploy script, or click a button in a dashboard.

---

## 9. Cost Model

### Monthly Cost Estimates for a Small Team

The platform is designed to start on free tiers and scale costs gradually with usage. The following estimates assume a small team (2 to 5 developers) in early production.

| Service | Free Tier | Paid Tier | Scales With |
|---------|-----------|-----------|-------------|
| **Supabase** | Generous free tier for development | $25/month Pro plan: 500 MB database, 1 GB file storage, 500K serverless function calls | Database size, storage volume, function invocations |
| **Vercel** | Free for personal projects | $20/month Pro plan: unlimited deployments, higher bandwidth | Traffic volume and build frequency |
| **OpenAI** | Pay-as-you-go only | ~$0.01 to $0.05 per image analysis (varies by model and image size) | Number of photos analyzed |
| **New Relic** | 100 GB/month data ingest, 1 full-access user | $0 for the free tier (sufficient for most startups) | Log volume and number of monitored services |
| **GitHub** | Free for public repositories | $4/user/month for private repositories | Team size |

### Estimated Monthly Total

| Stage | Approximate Cost |
|-------|-----------------|
| **Development and testing** | $0 (free tiers cover most needs) |
| **Early production** (small team, moderate usage) | $50 to $100/month |
| **Growth stage** (larger team, higher volume) | Scales linearly with usage |

### Why This Cost Model Works

- **No servers to size or provision**: All services are managed and auto-scale. There is no need to predict capacity or pay for idle resources.
- **No DevOps team required**: Routine operations (deployments, database backups, SSL certificates, CDN configuration) are handled by the managed services.
- **Pay-as-you-grow**: Every service starts free or near-free and costs increase proportionally with usage. There are no large upfront commitments or minimum contracts.
- **Predictable AI costs**: OpenAI charges per API call with published pricing. The cost of AI analysis is directly proportional to the number of photos processed, making it straightforward to forecast.

---

## Summary of Business Benefits

| Capability | Business Value |
|-----------|---------------|
| AI-powered image analysis | Replaces manual visual review; catches issues consistently and at scale |
| Configurable AI rules | Business teams can adjust detection behavior without waiting for a software release |
| AI eval framework | Rule changes are validated against known-good test cases before reaching production |
| Real-time updates | Field workers and reviewers see results as they happen -- no delays, no stale data |
| Preview environments per PR | Stakeholders can review features before they go live, with zero setup |
| Automated deployments | Every merge to main goes live without human intervention or special access |
| Row-level security | Data isolation between tenants is enforced at the database level, not just in application code |
| Single codebase (monorepo) | Shared data definitions eliminate inconsistencies between the two apps and the backend |
| Complete audit trails | Every action, every AI analysis, every status change is recorded for compliance and accountability |
| Offline-capable mobile app | Field workers can continue working without connectivity; data syncs automatically when back online |
| Multi-language support | English and Spanish out of the box; additional languages require translation files only, no code changes |
