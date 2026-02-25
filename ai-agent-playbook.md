# AI Agent Playbook — Supabase SaaS Platform Blueprint

> A step-by-step guide for an AI agent to scaffold, configure, and deploy a complete Supabase-powered SaaS platform from scratch. Domain-agnostic — adapt entity names and workflows to your specific business.

---

## Table of Contents

- [Phase 0: Information Gathering](#phase-0-information-gathering)
- [Phase 1: Monorepo Scaffold](#phase-1-monorepo-scaffold)
- [Phase 2: Shared Packages](#phase-2-shared-packages)
- [Phase 3: Frontend Apps](#phase-3-frontend-apps)
- [Phase 4: Supabase Setup](#phase-4-supabase-setup)
- [Phase 5: Edge Functions Pattern](#phase-5-edge-functions-pattern)
- [Phase 6: AI Pipeline Setup](#phase-6-ai-pipeline-setup)
- [Phase 7: Storage Setup](#phase-7-storage-setup)
- [Phase 8: Environment Variables](#phase-8-environment-variables)
- [Phase 9: Auth User Seeding](#phase-9-auth-user-seeding)
- [Phase 10: GitHub Repository Setup](#phase-10-github-repository-setup)
- [Phase 11: CI/CD Workflows](#phase-11-cicd-workflows)
- [Phase 12: Vercel Integration](#phase-12-vercel-integration)
- [Phase 13: Supabase GitHub Integration](#phase-13-supabase-github-integration)
- [Phase 14: E2E Test Infrastructure](#phase-14-e2e-test-infrastructure)
- [Phase 15: Observability](#phase-15-observability)
- [Phase 16: CLAUDE.md + Skills + AGENTS.md](#phase-16-claudemd--skills--agentsmd)
- [Definition of Done](#definition-of-done)

---

## Phase 0: Information Gathering

Before writing any code, gather these requirements from the user:

### Questions to Ask

1. **App names**: What should the admin portal and mobile app be called?
2. **Organization name**: What's the npm scope? (e.g., `@acme`)
3. **Tenancy model**: Multi-tenant (many orgs in one DB) or single-tenant (one org per deployment)?
4. **Core entities**: What are the main business objects? (e.g., orders, inspections, tickets)
5. **Photo workflows**: Does the mobile app need photo capture + AI analysis?
6. **User roles**: What roles exist? (e.g., Admin, Manager, Operator)
7. **Locales**: Which languages? (default: English + Spanish)
8. **External APIs**: Any third-party integrations needed? (e.g., maps, payments)

### Outputs

- Entity list with relationships
- User role hierarchy
- Tenancy decision (multi vs single)
- App naming conventions

---

## Phase 1: Monorepo Scaffold

### 1.1 Initialize Nx Workspace

```bash
npx create-nx-workspace@latest <project-name> \
  --preset=ts \
  --packageManager=pnpm \
  --nxCloud=skip

cd <project-name>
```

### 1.2 Install Core Dependencies

```bash
pnpm add -D @nx/vite @nx/react @nx/js vitest @vitest/coverage-v8 \
  @playwright/test typescript eslint prettier
```

### 1.3 Create Directory Structure

```
<project-name>/
├── apps/
│   ├── portal/          # Admin portal (React + Vite)
│   └── app/             # Mobile-first app (React + Vite)
├── packages/
│   ├── domain/          # Shared business logic
│   └── utils/           # Shared utilities
├── supabase/
│   ├── migrations/
│   ├── functions/
│   │   └── _shared/
│   ├── seed.sql
│   └── config.toml
├── scripts/
├── docs/
├── .github/
│   ├── workflows/
│   └── skills/
├── nx.json
├── pnpm-workspace.yaml
├── vitest.workspace.ts
└── package.json
```

### 1.4 Configure pnpm-workspace.yaml

```yaml
packages:
  - 'apps/*'
  - 'packages/*'
```

### 1.5 Configure nx.json

```json
{
  "$schema": "./node_modules/nx/schemas/nx-schema.json",
  "targetDefaults": {
    "build": { "dependsOn": ["^build"], "cache": true },
    "test": { "cache": true },
    "lint": { "cache": true },
    "e2e": { "cache": false }
  },
  "defaultBase": "main"
}
```

### 1.6 Add Root Scripts to package.json

```json
{
  "scripts": {
    "dev": "nx run-many -t dev --projects=@<org>/portal,@<org>/app",
    "portal": "nx dev @<org>/portal",
    "app": "nx dev @<org>/app",
    "build": "nx run-many -t build",
    "test": "nx run-many -t test",
    "lint": "nx run-many -t lint",
    "affected:lint": "nx affected -t lint",
    "affected:test": "nx affected -t test",
    "affected:e2e": "nx affected -t e2e",
    "graph": "nx graph",
    "supabase:start": "supabase start",
    "supabase:stop": "supabase stop",
    "supabase:reset": "supabase db reset && pnpm supabase:seed:auth",
    "supabase:functions": "supabase functions serve --env-file supabase/functions/.env",
    "supabase:functions:debug": "supabase functions serve --inspect --env-file supabase/functions/.env",
    "supabase:seed:auth": "npx tsx scripts/seed-auth.ts",
    "e2e": "nx run-many -t e2e",
    "e2e:portal": "nx e2e @<org>/portal-e2e",
    "e2e:app": "nx e2e @<org>/app-e2e",
    "e2e:local": "bash scripts/e2e-local.sh",
    "e2e:local:portal": "bash scripts/e2e-local.sh --portal",
    "e2e:local:app": "bash scripts/e2e-local.sh --app",
    "e2e:local:headed": "bash scripts/e2e-local.sh --headed"
  }
}
```

### Verification

- [ ] `pnpm install` succeeds
- [ ] `nx graph` shows workspace structure
- [ ] Directory structure matches plan

---

## Phase 2: Shared Packages

### 2.1 Domain Package (`packages/domain`)

Shared business types, constants, and enums used across apps and edge functions.

```
packages/domain/
├── src/
│   ├── index.ts
│   ├── <entity>/
│   │   ├── types.ts       # TypeScript interfaces
│   │   ├── constants.ts   # Status enums, magic values
│   │   └── index.ts       # Re-exports
│   └── auth/
│       ├── types.ts       # User, Role, Tenant types
│       └── index.ts
├── package.json
└── tsconfig.json
```

**package.json:**

```json
{
  "name": "@<org>/domain",
  "version": "0.0.1",
  "main": "src/index.ts",
  "types": "src/index.ts"
}
```

### 2.2 Utils Package (`packages/utils`)

Shared utilities: API helpers, formatters, validators.

```
packages/utils/
├── src/
│   ├── index.ts
│   ├── formatters.ts      # Date, currency, number formatters
│   ├── validators.ts      # Input validation helpers
│   └── api-helpers.ts     # Supabase query helpers
├── package.json
└── tsconfig.json
```

### Verification

- [ ] `import { OrderStatus } from '@<org>/domain'` resolves in both apps
- [ ] `pnpm build` succeeds for all packages

---

## Phase 3: Frontend Apps

### 3.1 Portal App (Admin)

Desktop-first admin portal. Initialize with Vite + React:

```bash
nx g @nx/react:application portal --directory=apps/portal \
  --bundler=vite --style=css --routing=true
```

**Key dependencies:**

```bash
cd apps/portal
pnpm add @supabase/supabase-js ag-grid-react ag-grid-community \
  react-router-dom @tanstack/react-query react-i18next i18next
```

**Structure:**

```
apps/portal/src/
├── main.tsx
├── App.tsx
├── shared/
│   ├── lib/
│   │   ├── supabase.ts          # Supabase client singleton
│   │   └── observability.ts     # New Relic setup
│   ├── components/
│   │   ├── ErrorBoundary.tsx
│   │   └── Layout.tsx
│   └── hooks/
│       └── useAuth.ts
├── features/
│   ├── <entity>/
│   │   ├── pages/
│   │   ├── components/
│   │   └── hooks/
│   └── settings/
├── i18n/
│   ├── en/
│   └── es/
└── test/
    └── setup.ts
```

### 3.2 Mobile App

Mobile-first PWA. Initialize similarly:

```bash
nx g @nx/react:application app --directory=apps/app \
  --bundler=vite --style=css --routing=true
```

**Key dependencies:**

```bash
cd apps/app
pnpm add @supabase/supabase-js react-router-dom \
  @tanstack/react-query react-i18next i18next
```

**Structure mirrors portal** but with mobile-specific patterns:
- Larger touch targets
- Bottom navigation
- Photo capture components
- Offline-first data patterns

### 3.3 Vite Configuration (Both Apps)

```typescript
// vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  server: {
    port: 5174,  // Portal: 5174, App: 5173
    host: true,
  },
  build: {
    outDir: 'dist',
    sourcemap: true,
  },
});
```

### 3.4 Supabase Client Singleton

```typescript
// shared/lib/supabase.ts
import { createClient } from '@supabase/supabase-js';
import type { Database } from '@<org>/domain';

export const supabase = createClient<Database>(
  import.meta.env.VITE_SUPABASE_URL,
  import.meta.env.VITE_SUPABASE_ANON_KEY,
);
```

### Verification

- [ ] `pnpm portal` serves at http://localhost:5174
- [ ] `pnpm app` serves at http://localhost:5173
- [ ] `pnpm dev` serves both simultaneously
- [ ] Supabase client connects to local instance

---

## Phase 4: Supabase Setup

### 4.1 Initialize Supabase

```bash
supabase init
```

### 4.2 Configure `supabase/config.toml`

Key settings to customize:

```toml
[api]
port = 54321

[db]
port = 54322
major_version = 17

[storage]
file_size_limit = "50MiB"

[storage.buckets.<project>-attachments]
public = false
file_size_limit = "50MiB"
allowed_mime_types = ["image/jpeg", "image/png", "image/webp"]

# Edge function JWT overrides (for server-to-server functions)
# [functions.<function-name>]
# verify_jwt = false
```

### 4.3 Decision Point: Multi-Tenant vs Single-Tenant

#### Multi-Tenant Schema

Use when many organizations share one database. Every business table has a `tenant_id`.

```sql
-- Core hierarchy
CREATE TABLE customers (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  name text NOT NULL,
  created_at timestamptz DEFAULT now()
);

CREATE TABLE tenants (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  customer_id uuid REFERENCES customers(id),
  name text NOT NULL,
  created_at timestamptz DEFAULT now()
);

CREATE TABLE facilities (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id uuid NOT NULL REFERENCES tenants(id),
  code text NOT NULL,
  name text NOT NULL,
  created_at timestamptz DEFAULT now()
);

-- App users linked to auth.users
CREATE TABLE app_users (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  auth_user_id uuid NOT NULL REFERENCES auth.users(id),
  email text NOT NULL,
  display_name text,
  created_at timestamptz DEFAULT now()
);

-- Tenant role assignments
CREATE TABLE user_tenant_roles (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id uuid NOT NULL REFERENCES app_users(id),
  tenant_id uuid NOT NULL REFERENCES tenants(id),
  role text NOT NULL CHECK (role IN ('admin', 'manager', 'operator')),
  UNIQUE(user_id, tenant_id)
);

-- Business entity example
CREATE TABLE orders (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id uuid NOT NULL REFERENCES tenants(id),
  facility_id uuid REFERENCES facilities(id),
  created_by uuid REFERENCES app_users(id),
  status text NOT NULL DEFAULT 'draft',
  created_at timestamptz DEFAULT now()
);

-- Indexes for RLS performance
CREATE INDEX idx_orders_tenant_id ON orders(tenant_id);
CREATE INDEX idx_user_tenant_roles_user_id ON user_tenant_roles(user_id);

-- RLS
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Tenant members can view orders"
  ON orders FOR SELECT TO authenticated
  USING (
    tenant_id IN (
      SELECT tenant_id FROM user_tenant_roles
      WHERE user_id = (
        SELECT id FROM app_users WHERE auth_user_id = (select auth.uid())
      )
    )
  );

-- Helper functions in private schema
CREATE SCHEMA IF NOT EXISTS private;

CREATE OR REPLACE FUNCTION private.get_app_user_id()
RETURNS uuid AS $$
  SELECT id FROM public.app_users WHERE auth_user_id = auth.uid()
$$ LANGUAGE sql STABLE SECURITY DEFINER;

CREATE OR REPLACE FUNCTION private.has_system_admin_role()
RETURNS boolean AS $$
  SELECT EXISTS (
    SELECT 1 FROM user_tenant_roles utr
    JOIN app_users au ON au.id = utr.user_id
    WHERE au.auth_user_id = auth.uid() AND utr.role = 'admin'
  )
$$ LANGUAGE sql STABLE SECURITY DEFINER;
```

#### Single-Tenant Schema

Use when each deployment serves one organization. Simpler hierarchy, no `tenant_id`.

```sql
-- Simpler hierarchy
CREATE TABLE organizations (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  name text NOT NULL,
  created_at timestamptz DEFAULT now()
);

-- Users directly linked to auth
CREATE TABLE profiles (
  id uuid PRIMARY KEY REFERENCES auth.users(id),
  email text NOT NULL,
  display_name text,
  role text NOT NULL DEFAULT 'member'
    CHECK (role IN ('admin', 'manager', 'member')),
  created_at timestamptz DEFAULT now()
);

-- Business entity — no tenant_id needed
CREATE TABLE orders (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  created_by uuid NOT NULL REFERENCES auth.users(id),
  status text NOT NULL DEFAULT 'draft',
  created_at timestamptz DEFAULT now()
);

-- Simpler RLS
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can view all orders"
  ON orders FOR SELECT TO authenticated
  USING (true);

CREATE POLICY "Users can create orders"
  ON orders FOR INSERT TO authenticated
  WITH CHECK (created_by = (select auth.uid()));
```

### 4.4 Create Initial Migration

```bash
pnpm supabase migration new initial_schema
```

Write the chosen schema into the generated file. Apply:

```bash
pnpm supabase:reset
```

### Verification

- [ ] `pnpm supabase:start` succeeds
- [ ] `pnpm supabase:reset` applies migrations and seeds
- [ ] Studio at http://127.0.0.1:54323 shows tables
- [ ] RLS policies block unauthenticated access

---

## Phase 5: Edge Functions Pattern

### 5.1 Shared Utilities

Create `supabase/functions/_shared/index.ts`:

```typescript
export const corsHeaders = {
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, OPTIONS',
  'Access-Control-Allow-Headers':
    'Content-Type, Authorization, X-Client-Info, Apikey',
};

export function jsonResponse(data: unknown, status = 200) {
  return new Response(JSON.stringify(data), {
    status,
    headers: { ...corsHeaders, 'Content-Type': 'application/json' },
  });
}

export function handleCors(req: Request) {
  if (req.method === 'OPTIONS') {
    return new Response(null, { status: 200, headers: corsHeaders });
  }
  return null;
}
```

### 5.2 Hello World Function

Create `supabase/functions/hello-world/index.ts`:

```typescript
import { Hono } from 'jsr:@hono/hono';
import 'jsr:@supabase/functions-js/edge-runtime.d.ts';

const app = new Hono().basePath('/hello-world');

app.get('/', (c) => c.json({ message: 'Hello from edge functions!' }));
app.get('/health', (c) => c.json({ status: 'healthy', timestamp: new Date().toISOString() }));

Deno.serve(app.fetch);
```

Create `supabase/functions/hello-world/deno.json`:

```json
{
  "imports": {
    "@platform/domain/": "../../../packages/domain/src/"
  }
}
```

### 5.3 Root Deno Configuration

Create `supabase/functions/deno.json`:

```json
{
  "compilerOptions": {
    "strict": true,
    "jsx": "react-jsx",
    "jsxImportSource": "npm:react"
  }
}
```

### 5.4 Environment File

Create `supabase/functions/.env`:

```bash
# Edge functions that call external APIs need explicit vars
OPENAI_API_KEY=sk-...
```

### Verification

- [ ] `pnpm supabase:functions` starts without errors
- [ ] `curl http://localhost:54321/functions/v1/hello-world` returns JSON
- [ ] `curl http://localhost:54321/functions/v1/hello-world/health` returns healthy

---

## Phase 6: AI Pipeline Setup

> Skip this phase if the app doesn't need AI-powered image analysis.

### 6.1 Enable Extensions

```sql
CREATE EXTENSION IF NOT EXISTS pgmq;
CREATE EXTENSION IF NOT EXISTS pg_cron;
CREATE EXTENSION IF NOT EXISTS pg_net;
```

### 6.2 Create Queue and Tables

```sql
-- Create the AI processing queue
SELECT pgmq.create('<type>_ai_queue');

-- Photos table
CREATE TABLE IF NOT EXISTS <entity>_photos (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  entity_id uuid NOT NULL REFERENCES <entities>(id),
  tenant_id uuid NOT NULL,  -- omit for single-tenant
  storage_key text NOT NULL,
  ai_status text DEFAULT 'pending'
    CHECK (ai_status IN ('pending', 'processing', 'completed', 'failed')),
  ai_summary text,
  ai_detected_types jsonb DEFAULT '[]',
  ai_last_processed_at timestamptz,
  ai_error text,
  created_at timestamptz DEFAULT now()
);
ALTER TABLE <entity>_photos ENABLE ROW LEVEL SECURITY;

-- AI audit log
CREATE TABLE IF NOT EXISTS <type>_ai_runs (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  photo_id uuid NOT NULL REFERENCES <entity>_photos(id),
  model text NOT NULL,
  prompt_tokens int,
  completion_tokens int,
  duration_ms int,
  status text NOT NULL CHECK (status IN ('completed', 'failed')),
  result jsonb,
  error text,
  created_at timestamptz DEFAULT now()
);

-- Dead letter queue
CREATE TABLE IF NOT EXISTS <type>_ai_dlq (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  original_msg_id bigint,
  photo_id uuid,
  error_message text,
  retry_count int DEFAULT 0,
  original_payload jsonb,
  failed_at timestamptz DEFAULT now()
);
```

### 6.3 Trigger and Cron Functions

See the `ai-detection-process` skill for complete function templates covering:
- `queue_<type>_ai_job()` — trigger to enqueue on photo INSERT
- `process_<type>_ai_queue_message()` — dequeue and call edge function via pg_net
- `apply_<type>_detection_results()` — apply AI results atomically

### 6.4 Cron Job

```sql
SELECT cron.schedule(
  'process-<type>-ai-queue',
  '* * * * *',
  $$SELECT process_<type>_ai_queue_message()$$
);
```

### 6.5 Edge Function

Create `supabase/functions/<type>-detector/index.ts` — fetches photo from storage, calls OpenAI Vision, applies results via RPC. See `ai-detection-process` skill for template.

Set `verify_jwt = false` in `config.toml` for this function.

### 6.6 Detection Configuration Tables

```sql
-- What to detect (configurable, not hardcoded)
CREATE TABLE IF NOT EXISTS detection_tags (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  tag_key text NOT NULL UNIQUE,
  display_name text NOT NULL,
  description text,
  confidence_guidance text,
  is_active boolean DEFAULT true
);

-- Business rules mapping detections to classifications
CREATE TABLE IF NOT EXISTS detection_rules (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  name text NOT NULL,
  tag_conditions jsonb NOT NULL,
  classification text NOT NULL,
  severity text DEFAULT 'info',
  is_active boolean DEFAULT true
);
```

### Verification

- [ ] Inserting a photo enqueues a PGMQ message
- [ ] Cron picks up the message and calls the edge function
- [ ] Edge function processes and writes results
- [ ] DLQ catches failures after retries

---

## Phase 7: Storage Setup

### 7.1 Bucket Configuration

In `supabase/config.toml`:

```toml
[storage.buckets.<project>-attachments]
public = false
file_size_limit = "50MiB"
allowed_mime_types = ["image/jpeg", "image/png", "image/webp"]
```

### 7.2 Storage Policies

```sql
CREATE POLICY "Authenticated users can upload"
  ON storage.objects FOR INSERT TO authenticated
  WITH CHECK (bucket_id = '<project>-attachments');

CREATE POLICY "Authenticated users can view"
  ON storage.objects FOR SELECT TO authenticated
  USING (bucket_id = '<project>-attachments');

CREATE POLICY "Authenticated users can update"
  ON storage.objects FOR UPDATE TO authenticated
  USING (bucket_id = '<project>-attachments');

CREATE POLICY "Authenticated users can delete"
  ON storage.objects FOR DELETE TO authenticated
  USING (bucket_id = '<project>-attachments');
```

### 7.3 Proxy Edge Functions

Create `save-attachment` and `get-attachment` edge functions that proxy storage access. Set `verify_jwt = false` for these if called server-to-server.

### Verification

- [ ] Upload via Supabase client succeeds
- [ ] Proxy edge function returns the file
- [ ] Unauthenticated requests are rejected

---

## Phase 8: Environment Variables

### 8.1 Local Development

`.env.local` at repo root:

```bash
VITE_SUPABASE_URL=http://127.0.0.1:54321
VITE_SUPABASE_ANON_KEY=<local-anon-key>
```

`supabase/functions/.env`:

```bash
OPENAI_API_KEY=sk-...
```

### 8.2 Production (GitHub Secrets)

| Secret | Purpose |
|--------|---------|
| `SUPABASE_ACCESS_TOKEN` | CLI authentication |
| `SUPABASE_PROJECT_REF` | Production project reference |
| `NEXT_PUBLIC_SUPABASE_URL` | Production Supabase URL |
| `NEXT_PUBLIC_SUPABASE_ANON_KEY` | Production anon key |
| `VERCEL_ORG_ID` | Vercel organization |
| `VERCEL_TOKEN` | Vercel deploy token |
| `VERCEL_PORTAL_PROJECT_ID` | Portal project ID |
| `VERCEL_APP_PROJECT_ID` | Mobile app project ID |
| `OPENAI_API_KEY` | For AI pipeline |

---

## Phase 9: Auth User Seeding

### 9.1 Seed Script

Create `scripts/seed-auth.ts`:

```typescript
import { createClient } from '@supabase/supabase-js';

const supabase = createClient(
  process.env.SUPABASE_URL || 'http://127.0.0.1:54321',
  process.env.SUPABASE_SERVICE_ROLE_KEY || '<local-service-role-key>',
  { auth: { autoRefreshToken: false, persistSession: false } }
);

const SEED_USERS = [
  { email: 'admin@acme-corp.com', password: 'Test1234!', user_metadata: { display_name: 'Admin User' } },
  { email: 'manager@acme-corp.com', password: 'Test1234!', user_metadata: { display_name: 'Manager User' } },
  { email: 'operator@acme-corp.com', password: 'Test1234!', user_metadata: { display_name: 'Operator User' } },
  { email: 'admin@test-co.com', password: 'Test1234!', user_metadata: { display_name: 'Test Co Admin' } },
];

async function seedUsers() {
  for (const user of SEED_USERS) {
    const { error } = await supabase.auth.admin.createUser({
      email: user.email,
      password: user.password,
      email_confirm: true,
      user_metadata: user.user_metadata,
    });
    if (error && !error.message.includes('already been registered')) {
      console.error(`Failed: ${user.email}:`, error.message);
    } else {
      console.log(`Created/exists: ${user.email}`);
    }
  }
}

seedUsers();
```

### 9.2 Link to App Users (Multi-Tenant)

In `seed.sql`:

```sql
INSERT INTO app_users (id, auth_user_id, email, display_name)
SELECT gen_random_uuid(), au.id, au.email, au.raw_user_meta_data->>'display_name'
FROM auth.users au
ON CONFLICT DO NOTHING;
```

### Verification

- [ ] `pnpm supabase:reset` creates auth users
- [ ] Can log in as seed users
- [ ] Data scoping works (tenant-scoped or user-scoped)

---

## Phase 10: GitHub Repository Setup

### 10.1 Create Repository

```bash
gh repo create <org>/<project-name> --private --source=. --push
```

### 10.2 Branch Protection

```bash
gh api repos/<org>/<project-name>/branches/main/protection -X PUT \
  -f required_status_checks='{"strict":true,"contexts":["Validate"]}' \
  -F enforce_admins=false \
  -f required_pull_request_reviews='{"required_approving_review_count":1}'
```

### 10.3 Add Secrets

```bash
gh secret set SUPABASE_ACCESS_TOKEN --body "<token>"
gh secret set SUPABASE_PROJECT_REF --body "<ref>"
# ... (see Phase 8 for full list)
```

---

## Phase 11: CI/CD Workflows

### 11.1 Main CI (`.github/workflows/ci.yml`)

Use the template from `docs/playbook/templates/ci.yml`. Structure:

```
validate → supabase-deploy → production-portal
                           ↗
                          → production-app
```

**Key points:**
- `validate`: lint + test affected (PR) or all (main push)
- `supabase-deploy`: `supabase db push` + `supabase functions deploy`
- Production jobs: Nx build → `.vercel/output/static/` → `vercel deploy --prebuilt --prod`
- `supabase db push` is idempotent — safe to re-run

### 11.2 Preview Comment (`.github/workflows/preview-comment.yml`)

Waits for Vercel preview deployments, then posts/updates a PR comment with links to Portal, App, and Supabase preview branch.

### 11.3 Preview Auth Seeding (`.github/workflows/seed-preview.yml`)

Waits for Supabase preview branch check to pass, then runs `pnpm supabase:seed:auth` against the preview database.

### 11.4 Cleanup (`.github/workflows/cleanup.yml`)

Logs PR closure. Supabase GitHub integration handles preview branch deletion automatically.

### Verification

- [ ] PR triggers CI validation
- [ ] PR gets preview comment with links
- [ ] Merge to main deploys Supabase then frontends

---

## Phase 12: Vercel Integration

1. Create Vercel projects for Portal and App
2. Set Root Directory per project (`apps/portal` or `apps/app`)
3. Add environment variables (Supabase URL, anon key)
4. Connect GitHub repository for auto preview deployments

---

## Phase 13: Supabase GitHub Integration

1. Go to Supabase Dashboard → Project Settings → Integrations
2. Connect GitHub repository
3. Enable preview branches

Each PR gets an isolated preview database with migrations and seed data applied automatically.

---

## Phase 14: E2E Test Infrastructure

### 14.1 Orchestration Script

Create `scripts/e2e-local.sh` that:
1. Verifies Docker is running
2. Starts Supabase if needed
3. Resets and seeds database
4. Starts edge functions in background
5. Runs Playwright tests
6. Cleans up on exit via `trap`

### 14.2 Playwright Configuration

Configure per app with:
- Global setup for pre-authentication
- Mobile Chrome device profile
- `workers: 1` in CI for stability
- `trace: 'on-first-retry'` for debugging
- Web server auto-start

### 14.3 Test Categories

| Category | App | Pattern |
|----------|-----|---------|
| Admin data grids | Portal | Read-only: assert seeded data |
| Admin configuration | Portal | Read-only: navigate and verify |
| Mobile workflows | App | Create data via UI |
| Photo capture | App | File chooser API (headless) |
| Auth flows | Both | Login/logout/redirect |

See the `pwa-e2e-testing` skill for detailed patterns.

### Verification

- [ ] `pnpm e2e:local` runs full suite
- [ ] Tests pass in CI

---

## Phase 15: Observability

### 15.1 New Relic Browser Agent

```bash
pnpm add @newrelic/browser-agent
```

### 15.2 Initialize with Logging Feature

Create `shared/lib/observability.ts`:

```typescript
import { BrowserAgent } from '@newrelic/browser-agent/loaders/browser-agent';
import { Logging } from '@newrelic/browser-agent/features/logging';

export function initObservability() {
  if (!import.meta.env.PROD || navigator.userAgent.includes('Playwright')) return;

  return new BrowserAgent({
    init: {
      distributed_tracing: { enabled: true },
      privacy: { cookies_enabled: true },
    },
    info: {
      beacon: 'bam.nr-data.net',
      errorBeacon: 'bam.nr-data.net',
      licenseKey: import.meta.env.VITE_NEW_RELIC_LICENSE_KEY,
      applicationID: import.meta.env.VITE_NEW_RELIC_APP_ID,
      sa: 1,
    },
    loader_config: {
      accountID: import.meta.env.VITE_NEW_RELIC_ACCOUNT_ID,
      trustKey: import.meta.env.VITE_NEW_RELIC_ACCOUNT_ID,
      agentID: import.meta.env.VITE_NEW_RELIC_APP_ID,
      licenseKey: import.meta.env.VITE_NEW_RELIC_LICENSE_KEY,
      applicationID: import.meta.env.VITE_NEW_RELIC_APP_ID,
    },
    features: [Logging], // Enables newrelic.log() and newrelic.wrapLogger()
  });
}
```

### 15.3 Structured Logger Utility

Create a typed logger that forwards structured logs to New Relic while preserving local dev output:

```typescript
// shared/lib/logger.ts
type LogLevel = 'debug' | 'info' | 'warn' | 'error' | 'trace';

interface LogPayload {
  feature: string;
  action: string;
  [key: string]: unknown;
}

const logger = {
  debug: (payload: LogPayload) => emit('debug', payload),
  info: (payload: LogPayload) => emit('info', payload),
  warn: (payload: LogPayload) => emit('warn', payload),
  error: (payload: LogPayload) => emit('error', payload),
};

function emit(level: LogLevel, payload: LogPayload) {
  const entry = { level, timestamp: new Date().toISOString(), ...payload };
  // Local dev: human-readable output
  if (import.meta.env.DEV) {
    const fn = level === 'error' ? console.error : level === 'warn' ? console.warn : console.log;
    fn(`[${level.toUpperCase()}] ${payload.feature}:${payload.action}`, payload);
    return;
  }
  // Production: forward to New Relic Logs via browser agent
  if (window.newrelic?.log) {
    window.newrelic.log(JSON.stringify(entry), { level });
  }
}

export { logger };
```

Alternatively, wrap an existing logger object with `newrelic.wrapLogger()`:

```typescript
// Wrap after agent initialization — each method becomes a log source
newrelic.wrapLogger(logger, 'info', { level: 'info' });
newrelic.wrapLogger(logger, 'warn', { level: 'warn' });
newrelic.wrapLogger(logger, 'error', { level: 'error', customAttributes: { critical: true } });
```

### 15.4 Error Reporting

```typescript
export function reportError(error: Error | string, context?: Record<string, unknown>) {
  const err = typeof error === 'string' ? new Error(error) : error;
  logger.error({ feature: context?.feature as string ?? 'unknown', action: 'unhandled_error', message: err.message, ...context });
  if (window.newrelic) window.newrelic.noticeError(err, context);
}
```

### 15.5 Edge Function Structured Logging

Edge functions use `console.log` with JSON payloads (captured by Supabase log drain):

```typescript
// supabase/functions/_shared/logger.ts
type LogLevel = 'debug' | 'info' | 'warn' | 'error';

export function log(level: LogLevel, payload: Record<string, unknown>) {
  const entry = { level, timestamp: new Date().toISOString(), ...payload };
  const fn = level === 'error' ? console.error : level === 'warn' ? console.warn : console.log;
  fn(JSON.stringify(entry));
}
```

Usage in edge functions:

```typescript
import { log } from '../_shared/logger.ts';

log('info', { fn: 'process-order', action: 'started', photoId, tenantId });
log('error', { fn: 'process-order', action: 'ai_call_failed', photoId, error: err.message, stack: err.stack });
```

### 15.6 Logging Patterns Summary

| Layer | Pattern | Destination |
|-------|---------|-------------|
| Frontend | `logger.info/warn/error()` → `newrelic.log()` | New Relic Logs |
| Edge Functions | `log('info', {...})` → JSON `console.log` | Supabase log drain → New Relic (optional) |
| Database | `RAISE NOTICE` / `RAISE WARNING` in PL/pgSQL | Supabase Postgres logs |

### Verification

- [ ] New Relic shows page views in production
- [ ] Structured logs appear in New Relic **Logs** UI (filter by `applicationID`)
- [ ] Errors include user context via `setCustomAttribute`
- [ ] E2E test users excluded (Playwright guard in `initObservability`)

---

## Phase 16: CLAUDE.md + Skills + AGENTS.md

Copy templates from `docs/playbook/templates/` to the repo root:

```bash
cp docs/playbook/templates/CLAUDE.md ./CLAUDE.md
cp docs/playbook/templates/AGENTS.md ./AGENTS.md
cp -r docs/playbook/skills/ .github/skills/
```

Customize: replace `<org>`, `<entity>`, seed user emails, port numbers, and add project-specific sections.

---

## Definition of Done

### Infrastructure
- [ ] `pnpm install` succeeds
- [ ] `pnpm supabase:start` + `pnpm supabase:reset` work
- [ ] `pnpm dev` starts both apps
- [ ] `pnpm supabase:functions` serves edge functions

### Code Quality
- [ ] `pnpm lint` passes
- [ ] `pnpm test` passes
- [ ] `pnpm build` succeeds

### E2E Tests
- [ ] `pnpm e2e:local` passes full suite
- [ ] Portal tests verify seeded data
- [ ] Mobile app tests create data via UI

### CI/CD
- [ ] PR triggers validation
- [ ] Preview environments created per PR
- [ ] Merge to main deploys: Supabase → Portal → App

### Observability
- [ ] New Relic initialized in production with `Logging` feature enabled
- [ ] Structured `logger` utility created (frontend + edge functions)
- [ ] Logs visible in New Relic **Logs** UI (filter by `applicationID`)
- [ ] Errors reported with user context (`setCustomAttribute`)
- [ ] E2E users excluded from analytics (Playwright guard)

### Documentation
- [ ] CLAUDE.md at repo root
- [ ] AGENTS.md at repo root
- [ ] Skills in `.github/skills/`
- [ ] i18n files for en + es
