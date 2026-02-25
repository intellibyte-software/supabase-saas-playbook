---
name: supabase-edge-functions
description: Guide for creating and deploying Supabase Edge Functions using Deno runtime. Use when building serverless functions, API endpoints, webhooks, or proxying external API calls. Covers CORS configuration, error handling, deployment patterns, Hono routing, and sharing code from a monorepo.
---

# Supabase Edge Functions

## CRITICAL: User Approval Required

**NEVER deploy edge functions without EXPLICIT user approval.**

### Required Workflow:

1. **Create local file first**: Write function to `supabase/functions/{name}/index.ts`
2. **Show the user**: Present the code and explain what it does
3. **STOP and wait**: Ask "Would you like me to deploy this edge function?"
4. **Only deploy after explicit approval**

---

## Deployment

### Preferred: Deploy via Supabase CLI

```bash
pnpm supabase functions deploy <function-name> --project-ref <PROJECT_REF>
```

**Why CLI is preferred:**
- Uploads real files — no truncation or payload size limits
- Automatically resolves imports from `_shared/` and includes them
- Reads `deno.json` import maps and bundles referenced files
- Works reliably for multi-file functions

**Notes:**
- `Docker is not running` warning is OK for deploy (Docker only needed for local serve)
- Deploy output shows each uploaded asset — verify shared files are included

### Fallback: Deploy via MCP

Only use MCP deploy for trivial single-file functions with no shared imports. It fails frequently with multi-file or large functions.

### Post-Deploy Verification

```bash
export SUPABASE_URL="https://<project-ref>.supabase.co"
export SUPABASE_ANON_KEY="<anon-key>"
export USER_JWT="<user-access-token>"

curl -sS "$SUPABASE_URL/functions/v1/<function-name>" \
   -H "apikey: $SUPABASE_ANON_KEY" \
   -H "Authorization: Bearer $USER_JWT" \
   -H "Content-Type: application/json" \
   --data '{"ping":true}'
```

### Pre-Deployment Checklist

1. Check existing functions with `mcp_supabase_list_edge_functions`
2. Create local file at `supabase/functions/{name}/index.ts`
3. **Ask user for approval**
4. Deploy via CLI or MCP after approval

### JWT Verification Setting

| Invocation Method | `verify_jwt` | Examples |
|---|---|---|
| **Cron jobs / PGMQ queue workers** | `false` | AI detectors, background processors |
| **Direct client calls (authenticated)** | `true` | User-facing APIs that need auth |
| **Webhooks / external callbacks** | `false` | Payment webhooks, third-party integrations |

**Rule of thumb**: If triggered by cron/PGMQ (not by authenticated users), set `verify_jwt: false`.

## Function Template

```typescript
import 'jsr:@supabase/functions-js/edge-runtime.d.ts';

const corsHeaders = {
   'Access-Control-Allow-Origin': '*',
   'Access-Control-Allow-Methods': 'GET, POST, PUT, DELETE, OPTIONS',
   'Access-Control-Allow-Headers':
      'Content-Type, Authorization, X-Client-Info, Apikey',
};

Deno.serve(async (req: Request) => {
   // CORS preflight — REQUIRED for all functions
   if (req.method === 'OPTIONS') {
      return new Response(null, { status: 200, headers: corsHeaders });
   }

   try {
      const data = { message: 'Hello' };
      return new Response(JSON.stringify(data), {
         headers: { ...corsHeaders, 'Content-Type': 'application/json' },
      });
   } catch (error) {
      return new Response(JSON.stringify({ error: error.message }), {
         status: 500,
         headers: { ...corsHeaders, 'Content-Type': 'application/json' },
      });
   }
});
```

## CORS Configuration (Mandatory)

Every function MUST handle OPTIONS and include CORS headers in ALL responses:

```typescript
if (req.method === 'OPTIONS') {
   return new Response(null, { status: 200, headers: corsHeaders });
}
```

## Import Rules

Use `npm:` or `jsr:` prefixes — never bare specifiers:

```typescript
// Correct
import { createClient } from 'npm:@supabase/supabase-js@2';
import 'jsr:@supabase/functions-js/edge-runtime.d.ts';

// Wrong — bare specifiers fail in Deno
import { createClient } from '@supabase/supabase-js';
```

## Error Handling

Always wrap entire function body in try/catch:

```typescript
Deno.serve(async (req: Request) => {
   if (req.method === 'OPTIONS') {
      return new Response(null, { status: 200, headers: corsHeaders });
   }

   try {
      // All logic inside try block
      return new Response(JSON.stringify({ success: true }), {
         headers: { ...corsHeaders, 'Content-Type': 'application/json' },
      });
   } catch (error) {
      console.error('Function error:', error);
      return new Response(JSON.stringify({ error: error.message }), {
         status: 500,
         headers: { ...corsHeaders, 'Content-Type': 'application/json' },
      });
   }
});
```

## Environment Variables

Pre-populated in all edge functions automatically:

- `SUPABASE_URL`
- `SUPABASE_ANON_KEY`
- `SUPABASE_SERVICE_ROLE_KEY`

```typescript
const supabaseUrl = Deno.env.get('SUPABASE_URL')!;
const serviceRoleKey = Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!;
const supabase = createClient(supabaseUrl, serviceRoleKey);
```

## Key Rules

| Rule | Details |
|------|---------|
| Use Deno.serve | Built-in Deno server, not Express |
| No cross-dependencies | Functions cannot import from other functions |
| Always handle OPTIONS | Required for browser CORS |
| Always try/catch | Wrap entire function body |
| Use npm:/jsr: imports | Never bare specifiers |
| Proxy external APIs | Never call external APIs from client |

## Hono Routing (Optional)

For functions with multiple endpoints:

```typescript
import { Hono } from 'jsr:@hono/hono';

const app = new Hono().basePath('/my-function');

app.get('/', (c) => c.json({ message: 'Hello' }));
app.get('/health', (c) => c.json({ status: 'healthy' }));
app.post('/process', async (c) => {
   const body = await c.req.json();
   return c.json({ result: 'processed' });
});

Deno.serve(app.fetch);
```

**Key points:**
- `.basePath("/function-name")` must match the function folder name
- Can split routes into separate files:

```
my-function/
├── index.ts           # Main app, mounts routes
├── deno.json          # Import map
└── routes/
    ├── process.ts     # /process endpoints
    └── health.ts      # /health endpoints
```

## Sharing Code from Monorepo

Edge functions can import from `packages/domain/` using import maps.

### Step 1: Per-function deno.json

```json
{
   "imports": {
      "@platform/domain/": "../../../packages/domain/src/"
   }
}
```

### Step 2: Import using the alias

```typescript
import { OrderStatus } from '@platform/domain/orders/status.ts';
```

**Note:** File extensions (`.ts`) are required for Deno imports.

### Step 3: Deploy via CLI

The CLI automatically resolves import maps and uploads referenced files.

## Long-Running Functions (Background Processing)

For functions that take > 10 seconds, use `EdgeRuntime.waitUntil()`:

```typescript
declare const EdgeRuntime: { waitUntil(promise: Promise<unknown>): void };

async function processInBackground(runId: string, supabase: any): Promise<void> {
   try {
      // Heavy processing...
      await supabase.from('runs').update({ status: 'completed' }).eq('id', runId);
   } catch (error) {
      console.error('Background processing failed:', error);
      await supabase.from('runs').update({ status: 'failed', error_message: String(error) }).eq('id', runId);
   }
}

Deno.serve(async (req: Request) => {
   const { run_id } = await req.json();
   await supabase.from('runs').update({ status: 'running' }).eq('id', run_id);

   EdgeRuntime.waitUntil(processInBackground(run_id, supabase));

   return new Response(JSON.stringify({ run_id, status: 'running' }), {
      status: 202,
      headers: { ...corsHeaders, 'Content-Type': 'application/json' },
   });
});
```

### Do NOT use self-invocation

The old pattern of `fetch()` to self with `_background: true` is unreliable — the isolate can shut down after returning the response.

### Pair with Realtime

```sql
ALTER PUBLICATION supabase_realtime ADD TABLE runs;
```

```typescript
const channel = supabase
   .channel('runs')
   .on('postgres_changes', { event: 'UPDATE', schema: 'public', table: 'runs' }, (payload) => {
      if (payload.new.status === 'completed') onCompleted(payload.new);
      if (payload.new.status === 'failed') onFailed(payload.new);
   })
   .subscribe();
```

### Checklist

- [ ] `declare const EdgeRuntime` type declaration at top
- [ ] `EdgeRuntime.waitUntil(...)` called before returning 202
- [ ] Background function updates DB to `completed` or `failed`
- [ ] Migration adds table to `supabase_realtime` publication
- [ ] Client subscribes to `postgres_changes` for status updates
