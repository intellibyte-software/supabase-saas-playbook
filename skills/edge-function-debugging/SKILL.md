---
name: edge-function-debugging
description: Guide for debugging Supabase Edge Functions locally and against deployed services. Use when troubleshooting edge function errors, unexpected behavior, or when investigating bug reports. Covers starting the local functions server, capturing console output, Chrome DevTools debugging, and testing against remote Supabase instances.
---

# Edge Function Debugging

## Debugging Workflow

1. Start the functions server as a **background process** to capture logs
2. Make test requests from a **separate terminal**
3. Read logs from the server terminal using its ID
4. Analyze log output to identify the bug
5. Fix and verify

## Starting the Functions Server

### Local Supabase

Start the server with `isBackground: true` to get a terminal ID for log retrieval:

```typescript
run_in_terminal({
  command: "pnpm supabase functions serve --env-file supabase/functions/.env",
  explanation: "Starting supabase functions server as background process",
  isBackground: true
})
// Returns: terminal ID like "e5aed5cd-c7ac-4c3c-93a8-558a804343c5"
```

Save this terminal ID — it's required to read logs later.

### With Chrome DevTools

For step-through debugging with breakpoints:

```bash
pnpm supabase functions serve --inspect --env-file supabase/functions/.env
```

Then open `chrome://inspect` in Chrome and add `127.0.0.1:8083` as a network target. You can set breakpoints, inspect variables, and step through code.

## Making Test Requests

**Critical**: Run test requests from a separate terminal to avoid killing the server:

```bash
# Local Supabase
curl -sS "http://localhost:54321/functions/v1/<function-name>" \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <anon-key-or-user-jwt>" \
  -H "apikey: <anon-key>" \
  --data '{"key": "value"}'
```

### Testing Against Deployed Services

When debugging functions that call external APIs or a remote Supabase instance:

```bash
# Serve locally but point to remote Supabase
pnpm supabase functions serve \
  --env-file supabase/functions/.env.remote \
  --no-verify-jwt
```

Create `.env.remote` with production/staging connection strings:

```bash
# supabase/functions/.env.remote
SUPABASE_URL=https://<project-ref>.supabase.co
SUPABASE_ANON_KEY=<remote-anon-key>
SUPABASE_SERVICE_ROLE_KEY=<remote-service-role-key>
OPENAI_API_KEY=<api-key>  # For AI functions
```

This lets you run the function code locally while accessing real data and services.

## Reading Server Logs

Use `get_terminal_output` with the server's terminal ID:

```typescript
get_terminal_output({ id: "e5aed5cd-c7ac-4c3c-93a8-558a804343c5" })
```

Logs include timestamps and `[Info]` tags for each `console.log`:

```
2026-01-25T17:34:15.168136456Z [Info]     Processing photo: abc123
2026-01-25T17:34:15.168624458Z [Info]     Detection result: { tags: [...] }
```

## Adding Debug Logging

```typescript
function processData(input: unknown): unknown {
  console.log("processData called");
  console.log("  Input:", JSON.stringify(input));

  const result = transform(input);
  console.log("  Result:", JSON.stringify(result));

  return result;
}
```

**Tips:**
- Use `JSON.stringify()` to see exact values including whitespace
- Log before and after key operations
- Include variable names in log messages for clarity
- Use indentation to show nesting/hierarchy
- For AI functions, log the prompt and response token counts

## Common Issues

### Terminal ID Not Working

`get_terminal_output` only works with terminal IDs from `run_in_terminal` with `isBackground: true`. It cannot read from pre-existing terminals.

### Server Stops When Running Commands

Running a foreground command in the same terminal kills the background server. Always run test requests from a separate terminal.

### Port Already in Use

If you see "context canceled" or port conflicts, another server instance is running. Kill existing processes first:

```bash
lsof -ti:54321 | xargs kill -9  # Kill process on Supabase API port
```

### Missing Logs

If logs don't appear:
- Ensure `console.log` statements are inside the request handler
- Check the function was actually invoked (verify URL path matches function name)
- Logs only appear after the request completes
- For `EdgeRuntime.waitUntil()` functions, logs from background processing may appear later

### Edge Function Returns 401

- Check `verify_jwt` setting in `supabase/config.toml`
- For cron/queue-triggered functions, set `verify_jwt = false`
- Ensure request includes both `apikey` and `Authorization` headers
