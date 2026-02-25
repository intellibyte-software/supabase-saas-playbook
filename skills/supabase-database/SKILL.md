---
name: supabase-database
description: Guide for Supabase database operations including migrations, baselining, Row Level Security (RLS), and query best practices. Use when creating tables, writing migrations, consolidating schema baselines, setting up RLS policies, configuring authentication, or writing Supabase queries. Covers local dev workflow, DDL operations, multi-tenant and single-tenant security patterns, and data safety.
---

# Supabase Database Operations

## CRITICAL: User Approval Required

**NEVER deploy migrations or execute data-modifying SQL without EXPLICIT user approval.**

### Forbidden Without Approval:

- `mcp_supabase_apply_migration` - Deploys schema changes to production
- `mcp_supabase_execute_sql` with INSERT/UPDATE/DELETE - Modifies production data

### Required Workflow:

1. **Create local file first**: Write migration to `supabase/migrations/YYYYMMDDHHMMSS_name.sql`
2. **Show the user**: Present the migration SQL and explain what it does
3. **STOP and wait**: Ask "Would you like me to deploy this migration?"
4. **Only deploy after explicit approval**: User must say "yes", "approved", "deploy it", etc.

**If user doesn't explicitly approve, DO NOT DEPLOY. Just leave the local file.**

---

## Local Development Workflow

### Starting Supabase

```bash
pnpm supabase:start      # Start Postgres, Auth, Storage, Edge Runtime, Studio
pnpm supabase:reset       # Reset DB + run all migrations + load seed data
```

### Iterating on Schema

```bash
# Create a new migration
pnpm supabase migration new <descriptive_name>
# Edit the generated file in supabase/migrations/
# Apply it:
pnpm supabase:reset       # Full reset (recommended during dev)
supabase migration up      # Incremental apply (no seed, faster)
```

### Verifying

- Open Studio at http://127.0.0.1:54323 to inspect tables
- Run the app (`pnpm dev`) to confirm queries work
- Run tests (`pnpm test`) to validate

---

## Migrations

### File Format

```
supabase/migrations/YYYYMMDDHHMMSS_<descriptive_name>.sql
```

### Migration Header Template

```sql
/*
  # Migration Title Here

  Plain English explanation of what this migration does.

  ## New Tables
  - `table_name` - Description of table purpose
    - `column_name` (type) - Column description

  ## Security
  - Enable RLS on `table_name`
  - Add policies for authenticated users

  ## Notes
  - Any important considerations
*/
```

### Migration Rules

- Use `IF EXISTS` / `IF NOT EXISTS` for idempotency
- Never use transaction control (`BEGIN`, `COMMIT`, `ROLLBACK`)
- One migration per logical change
- Each migration must have a unique timestamp
- Never edit an already-applied migration — always create a new one

### What Goes Where

| Content | Location |
|---------|----------|
| Schema (tables, enums, functions, triggers, indexes, RLS) | Migration file |
| Static config (roles, system lookup types, queue names) | `seed.sql` |
| Test/dev data (users, tenants, sample records) | `seed.sql` |

---

## Baselining (Schema Consolidation)

When migration count exceeds ~30 files, consolidate into a single baseline.

### When to Consolidate

- Migration count > ~30 files
- `pnpm supabase:reset` becomes slow
- Preparing for a major version release

### Step-by-Step

```bash
# 1. Link to production
supabase link --project-ref <SUPABASE_PROJECT_REF>

# 2. Dump current production schema
supabase db dump --linked > supabase/migrations/$(date +%Y%m%d)000000_baseline.sql

# 3. Delete old migration files (keep backups)
rm supabase/migrations/20260131*.sql  # etc.

# 4. Mark each deleted migration as applied in production
supabase migration repair --status applied 20260131000000
supabase migration repair --status applied 20260201100000
# Repeat for each deleted timestamp

# 5. Test locally
pnpm supabase:reset

# 6. Verify the app works
pnpm dev
```

### Naming Convention

```
YYYYMMDD000000_baseline.sql
```

The `000000` time ensures it sorts before any migrations created that day.

---

## Row Level Security (RLS)

### Enable RLS on Every Table

```sql
ALTER TABLE tablename ENABLE ROW LEVEL SECURITY;
```

After enabling, the table is locked down by default (RESTRICTIVE).

### Policy Structure by Operation

| Operation | Requires USING | Requires WITH CHECK |
| --------- | -------------- | ------------------- |
| SELECT    | Always         | Never               |
| INSERT    | Never          | Always              |
| UPDATE    | Always         | Always              |
| DELETE    | Always         | Never               |

### Policy Rules

- Never use `FOR ALL` — create separate policies for each operation
- Never use `USING (true)` — defeats RLS purpose
- Use `auth.uid()` for auth checks
- Use descriptive names in double quotes
- Use `TO authenticated` to scope policies to logged-in users

### Single-Tenant RLS Pattern

When each user owns their own data (no tenant hierarchy):

```sql
-- SELECT
CREATE POLICY "Users can view own records"
  ON tablename FOR SELECT TO authenticated
  USING (auth.uid() = user_id);

-- INSERT
CREATE POLICY "Users can insert own records"
  ON tablename FOR INSERT TO authenticated
  WITH CHECK (auth.uid() = user_id);

-- UPDATE
CREATE POLICY "Users can update own records"
  ON tablename FOR UPDATE TO authenticated
  USING (auth.uid() = user_id)
  WITH CHECK (auth.uid() = user_id);

-- DELETE
CREATE POLICY "Users can delete own records"
  ON tablename FOR DELETE TO authenticated
  USING (auth.uid() = user_id);
```

### Multi-Tenant RLS Pattern

When organizations share a database with `tenant_id` isolation:

```sql
-- Helper: get user's tenant IDs (cache with subquery for performance)
CREATE POLICY "Tenant members can view records"
  ON tablename FOR SELECT TO authenticated
  USING (
    tenant_id IN (
      SELECT (select auth.uid()) -- subquery caches per-statement
      FROM user_tenant_roles
      WHERE user_id = (select auth.uid())
    )
  );

-- INSERT with tenant check
CREATE POLICY "Tenant members can insert records"
  ON tablename FOR INSERT TO authenticated
  WITH CHECK (
    tenant_id IN (
      SELECT tenant_id FROM user_tenant_roles
      WHERE user_id = (select auth.uid())
    )
  );
```

**Performance tip**: Use `(select auth.uid())` instead of bare `auth.uid()` — the subquery form is evaluated once per statement, not once per row.

### RLS Testing Checklist

- [ ] Authenticated users access only their allowed data
- [ ] Unauthenticated users cannot access protected data
- [ ] Users cannot access other tenants'/users' data
- [ ] Edge cases in policy conditions work correctly

---

## Query Best Practices

### Use maybeSingle() for Optional Results

```typescript
// Correct — returns null if no rows
const { data } = await supabase
   .from('table')
   .select('*')
   .eq('id', id)
   .maybeSingle();

// Wrong — throws error if no rows
const { data } = await supabase.from('table').select('*').eq('id', id).single();
```

## Client Setup

```typescript
import { createClient } from '@supabase/supabase-js';
import type { Database } from './types';

export const supabase = createClient<Database>(
   import.meta.env.NEXT_PUBLIC_SUPABASE_URL,
   import.meta.env.NEXT_PUBLIC_SUPABASE_ANON_KEY,
);
```

## Authentication

### Default Pattern: Email/Password

```typescript
// Sign up
const { data, error } = await supabase.auth.signUp({ email, password });

// Sign in
const { data, error } = await supabase.auth.signInWithPassword({ email, password });

// Sign out
await supabase.auth.signOut();
```

### Auth State Listener (Avoid Deadlocks)

```typescript
// Correct — wrap async code in IIFE
supabase.auth.onAuthStateChange((event, session) => {
   (async () => {
      // async code here
   })();
});

// Wrong — async callback causes deadlocks
supabase.auth.onAuthStateChange(async (event, session) => {
   // This can deadlock!
});
```

### Auth Rules

- Use built-in `auth.users` — never create custom auth tables
- Email confirmation disabled by default in local dev
- Never use magic links/social providers unless explicitly requested

## Views

### Always Use SECURITY INVOKER

**Views MUST be created with `security_invoker = true`** to enforce RLS based on the querying user.

```sql
-- CORRECT
CREATE VIEW my_view
WITH (security_invoker = true)
AS SELECT * FROM my_table;

-- WRONG — bypasses RLS
CREATE VIEW my_view AS SELECT * FROM my_table;
```

After creating, grant permissions:

```sql
GRANT SELECT ON my_view TO authenticated;
```

## Data Safety

### User Foreign Keys

For multi-tenant apps with an `app_users` table:

```sql
-- Reference app_users, not auth.users
created_by uuid REFERENCES app_users(id) ON DELETE SET NULL
```

For single-tenant apps, reference `auth.users` directly:

```sql
created_by uuid REFERENCES auth.users(id) ON DELETE SET NULL
```

### Storage Bucket Policies

```sql
CREATE POLICY "Users can insert attachments" ON storage.objects
FOR INSERT TO authenticated WITH CHECK (bucket_id = '<bucket-name>');

CREATE POLICY "Users can view attachments" ON storage.objects
FOR SELECT TO authenticated USING (bucket_id = '<bucket-name>');

CREATE POLICY "Users can update attachments" ON storage.objects
FOR UPDATE TO authenticated USING (bucket_id = '<bucket-name>');

CREATE POLICY "Users can delete attachments" ON storage.objects
FOR DELETE TO authenticated USING (bucket_id = '<bucket-name>');
```

### Column Defaults

```sql
is_active boolean default false
created_at timestamptz default now()
count integer default 0
```

### Priorities

1. **Never lose user data** — highest priority
2. Avoid destructive operations (`DROP`, data-losing `DELETE`)
3. Use meaningful defaults for columns
4. Implement foreign key constraints
5. Add indexes for frequently queried columns
