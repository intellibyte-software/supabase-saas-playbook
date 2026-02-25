# AGENTS.md

> Instructions for AI agents (GitHub Copilot, Claude Code, etc.) working on this repository.

## Project Overview

This is an Nx monorepo containing a Supabase-powered SaaS platform with two frontend apps and shared packages.

## Repository Structure

```
├── apps/
│   ├── portal/            # Admin portal (React + Vite)
│   └── app/               # Mobile-first app (React + Vite)
├── packages/
│   ├── domain/            # Shared business logic, types, constants
│   └── utils/             # Shared utilities (API helpers, formatters)
├── supabase/
│   ├── migrations/        # Database migrations (run in timestamp order)
│   ├── functions/         # Deno edge functions
│   │   ├── _shared/       # Shared utilities (CORS, helpers)
│   │   └── <fn-name>/     # One directory per function
│   ├── seed.sql           # Dev/test seed data
│   └── config.toml        # Local Supabase configuration
├── scripts/               # Build, seed, and orchestration scripts
├── .github/
│   ├── workflows/         # CI/CD pipelines
│   └── skills/            # AI agent skill definitions
└── docs/                  # Architecture and playbook documentation
```

## Development Commands

```bash
pnpm install               # Install all dependencies
pnpm dev                   # Start both apps (portal:5174, app:5173)
pnpm build                 # Build all projects
pnpm test                  # Run all unit tests
pnpm lint                  # Lint all projects
```

## Supabase Commands

```bash
pnpm supabase:start        # Start local Supabase (requires Docker)
pnpm supabase:stop         # Stop local Supabase
pnpm supabase:reset        # Reset DB + run migrations + seed data + create auth users
pnpm supabase:functions    # Serve edge functions locally with hot reload
```

## Database Migrations

- File format: `supabase/migrations/YYYYMMDDHHMMSS_<name>.sql`
- Create: `pnpm supabase migration new <name>`
- Apply locally: `pnpm supabase:reset`
- **Never edit existing migrations** — always create new ones
- Use `IF NOT EXISTS` / `IF EXISTS` for idempotency
- Always enable RLS on new tables

## Testing

```bash
pnpm test                  # Unit tests (Vitest)
pnpm e2e:local             # Full E2E suite (starts infrastructure)
pnpm e2e:local:portal      # Portal E2E only
pnpm e2e:local:app         # Mobile app E2E only
pnpm affected:test         # Test only changed projects
pnpm affected:e2e          # E2E only changed apps
```

## Code Quality Rules

1. **Fix ESLint errors** on every file you touch — run `pnpm lint`
2. **i18n is mandatory** — no hardcoded user-facing strings; add keys to `en/` and `es/` translation files
3. **Tests alongside features** — every new feature or bug fix needs a test
4. **RLS on every table** — `ALTER TABLE ... ENABLE ROW LEVEL SECURITY` + policies
5. **Never deploy migrations** without explicit approval
6. **Edge function CORS** — every function must handle OPTIONS and include CORS headers

## Skills

Detailed guidance for specific domains is in `.github/skills/`:

| Skill | Use When |
|-------|----------|
| `supabase-database` | Creating tables, migrations, RLS policies |
| `supabase-edge-functions` | Building or deploying edge functions |
| `supabase-postgres-best-practices` | Optimizing queries or schema design |
| `ai-detection-process` | Adding AI-powered image detection |
| `edge-function-debugging` | Troubleshooting edge function errors |
| `pwa-e2e-testing` | Writing Playwright E2E tests for the mobile app |
| `vercel-composition-patterns` | React component architecture decisions |

## CI/CD Pipeline

Merges to `main` trigger: `validate → supabase-deploy → production deployments`

- Supabase deploys **before** frontends (migrations must be applied first)
- PRs get automatic preview environments (Vercel + Supabase preview branches)
- Nx affected detection runs only changed projects in PRs
