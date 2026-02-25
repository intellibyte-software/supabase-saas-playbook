# Supabase SaaS Playbook

A comprehensive, domain-agnostic blueprint for building production-ready SaaS applications with **Supabase**, **Vercel**, and **React**. Covers both **multi-tenant** and **single-tenant** architectures.

Use this repository as a template to scaffold new projects — or as a reference for AI agents (via [GitMCP](https://gitmcp.io)) and engineering staff.

## Quick Start

1. Click **"Use this template"** on GitHub to create a new repo
2. Copy `templates/CLAUDE.md` → `./CLAUDE.md` in your new repo
3. Copy `templates/AGENTS.md` → `./AGENTS.md`
4. Copy `skills/*` → `.github/skills/`
5. Copy `templates/ci.yml` → `.github/workflows/ci.yml`
6. Follow the [AI Agent Playbook](ai-agent-playbook.md) phase by phase

## Documentation

### Playbook

| Document | Audience | Description |
|----------|----------|-------------|
| [AI Agent Playbook](ai-agent-playbook.md) | AI agents / engineers | Phase-by-phase blueprint to scaffold a new Supabase SaaS project (16 phases) |
| [Architecture](architecture.md) | Engineers | Technical architecture reference — data model, AI pipeline, CI/CD, observability |
| [Executive Summary](executive-summary.md) | Non-technical stakeholders | High-level overview of what the system does and why |

### Templates

Copy these into your new project:

| Template | Target Location | Description |
|----------|----------------|-------------|
| [CLAUDE.md](templates/CLAUDE.md) | `./CLAUDE.md` | Claude Code instructions — local stack, migrations, E2E tests, AI architecture |
| [AGENTS.md](templates/AGENTS.md) | `./AGENTS.md` | GitHub Copilot / AI agent instructions — commands, workflow, code quality |
| [ci.yml](templates/ci.yml) | `.github/workflows/ci.yml` | GitHub Actions CI/CD — validate, Supabase deploy, Vercel production deploy |

### Skills

Copy the entire `skills/` directory to `.github/skills/` in your project. Skills provide AI agents with specialized domain knowledge.

| Skill | Description |
|-------|-------------|
| [supabase-database](skills/supabase-database/SKILL.md) | Local dev, migrations, baselining, RLS patterns (multi-tenant & single-tenant) |
| [supabase-edge-functions](skills/supabase-edge-functions/SKILL.md) | Edge function development, deployment, CORS, Hono routing, monorepo sharing |
| [supabase-postgres-best-practices](skills/supabase-postgres-best-practices/SKILL.md) | Query performance, indexes, connection pooling, RLS optimization |
| [ai-detection-process](skills/ai-detection-process/SKILL.md) | AI vision pipeline — PGMQ queues, cron jobs, OpenAI Vision, Realtime broadcasts |
| [edge-function-debugging](skills/edge-function-debugging/SKILL.md) | Local debugging workflow, Chrome DevTools, testing against remote services |
| [pwa-e2e-testing](skills/pwa-e2e-testing/SKILL.md) | Playwright E2E for mobile-first PWAs — timing, photo uploads, parallel isolation |
| [skill-creator](skills/skill-creator/SKILL.md) | Meta-skill: how to create new skills for AI agents |
| [vercel-composition-patterns](skills/vercel-composition-patterns/SKILL.md) | React composition patterns — compound components, state management, React 19 APIs |

## Tech Stack

| Layer | Technology |
|-------|-----------|
| **Frontend** | React + TypeScript, Vite, TanStack Router/Query, Tailwind CSS, shadcn/ui |
| **Backend** | Supabase (Postgres 17, Auth, Storage, Realtime, Edge Functions) |
| **Edge Runtime** | Deno + Hono framework |
| **AI** | OpenAI Vision API (gpt-4o / gpt-4o-mini) |
| **Monorepo** | Nx + pnpm workspaces |
| **CI/CD** | GitHub Actions → Supabase CLI + Vercel prebuilt deploy |
| **Observability** | New Relic (browser agent + structured logging) |
| **E2E Testing** | Playwright |

## AI Agent Access

### GitMCP (recommended)

Add this as an MCP server in your AI tool (Claude Code, Cursor, etc.):

```
https://gitmcp.io/intellibyte-software/supabase-saas-playbook
```

### GitHub MCP Server

If you already use [GitHub's official MCP server](https://github.com/github/github-mcp-server), it can read files from this repo directly.

## Architecture Highlights

- **Multi-tenant**: `tenant_id` on every table, junction-based user-tenant roles, RLS with `(SELECT auth.uid())` caching
- **Single-tenant**: Direct `user_id` ownership, `profiles` table extending `auth.users`, simpler RLS
- **AI Pipeline**: Photo INSERT → DB trigger → PGMQ → pg_cron → Edge Function → OpenAI Vision → apply results RPC → Realtime broadcast
- **Observability**: New Relic browser agent with `Logging` feature for structured log forwarding, edge function JSON logging via Supabase log drain

## License

MIT
