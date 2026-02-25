---
name: ai-detection-process
description: Guide for adding new AI-powered image detection processes. Use when implementing OpenAI Vision-based detection (e.g., damage detection, quality inspection, object classification) that follows the queue-based async pattern with PGMQ, cron jobs, and Deno edge functions. Covers the full pipeline from photo insert trigger to real-time result delivery.
---

# AI Detection Process

Add new AI-powered image detection following the established queue-based async architecture.

## Architecture Flow

```
Photo Insert → Trigger → PGMQ Queue → Cron Job → Edge Function → OpenAI Vision
                                                       ↓
                                              Apply Results RPC
                                                       ↓
                                            Update Photo + Audit Log
                                                       ↓
                                            Realtime Status Broadcast
```

## Implementation Checklist

1. **Database Schema** — Photo table columns, audit table, DLQ table
2. **PGMQ Queue** — Create queue for async processing
3. **Database Functions** — Trigger, apply results RPC, queue processor
4. **Cron Job** — Schedule queue consumption
5. **Edge Function** — Deno function calling OpenAI Vision
6. **Deploy** — Deploy edge function via CLI

## Step 1: Database Schema

Apply a migration with these components:

### Photo Table Columns

Add AI columns to your photo/media table:

```sql
ALTER TABLE <entity>_photos ADD COLUMN IF NOT EXISTS ai_summary text;
ALTER TABLE <entity>_photos ADD COLUMN IF NOT EXISTS ai_detected_types jsonb DEFAULT '[]';
ALTER TABLE <entity>_photos ADD COLUMN IF NOT EXISTS ai_status text DEFAULT 'pending'
  CHECK (ai_status IN ('pending', 'processing', 'completed', 'failed'));
ALTER TABLE <entity>_photos ADD COLUMN IF NOT EXISTS ai_last_processed_at timestamptz;
ALTER TABLE <entity>_photos ADD COLUMN IF NOT EXISTS ai_error text;
```

### Audit Table

```sql
CREATE TABLE IF NOT EXISTS <type>_ai_runs (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  photo_id uuid NOT NULL REFERENCES <entity>_photos(id),
  tenant_id uuid NOT NULL,  -- omit for single-tenant
  model text NOT NULL,
  prompt_tokens int,
  completion_tokens int,
  duration_ms int,
  status text NOT NULL CHECK (status IN ('completed', 'failed')),
  result jsonb,
  error text,
  created_at timestamptz DEFAULT now()
);
ALTER TABLE <type>_ai_runs ENABLE ROW LEVEL SECURITY;
```

### Dead Letter Queue

```sql
CREATE TABLE IF NOT EXISTS <type>_ai_dlq (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  original_msg_id bigint,
  photo_id uuid,
  tenant_id uuid,  -- omit for single-tenant
  error_message text,
  retry_count int DEFAULT 0,
  original_payload jsonb,
  failed_at timestamptz DEFAULT now()
);
```

### PGMQ Queue

```sql
SELECT pgmq.create('<type>_ai_queue');
```

## Step 2: Database Functions

| Function | Purpose |
|----------|---------|
| `queue_<type>_ai_job` | Trigger function — enqueue photo on INSERT |
| `apply_<type>_detection_results` | Atomically apply AI results, insert audit log |
| `process_<type>_ai_queue_message` | Read queue, call edge function via pg_net |
| `trigger_<type>_ai_queue_worker` | Cron-invoked function to spawn workers |
| `get_<type>_detection_types` | Get detection type config with i18n labels |

### Trigger Function Pattern

```sql
CREATE OR REPLACE FUNCTION queue_<type>_ai_job()
RETURNS trigger AS $$
BEGIN
  PERFORM pgmq.send(
    '<type>_ai_queue',
    jsonb_build_object(
      'photo_id', NEW.id,
      'tenant_id', NEW.tenant_id,  -- omit for single-tenant
      'storage_key', NEW.storage_key
    )
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER trg_queue_<type>_ai
  AFTER INSERT ON <entity>_photos
  FOR EACH ROW EXECUTE FUNCTION queue_<type>_ai_job();
```

### Queue Processor Pattern

```sql
CREATE OR REPLACE FUNCTION process_<type>_ai_queue_message()
RETURNS void AS $$
DECLARE
  v_msg record;
BEGIN
  -- Read one message with 60s visibility timeout
  SELECT * INTO v_msg FROM pgmq.read('<type>_ai_queue', 60, 1);
  IF v_msg IS NULL THEN RETURN; END IF;

  -- Call edge function via pg_net
  PERFORM net.http_post(
    url := current_setting('app.settings.supabase_url') || '/functions/v1/<type>-detector',
    headers := jsonb_build_object(
      'Content-Type', 'application/json',
      'Authorization', 'Bearer ' || current_setting('app.settings.service_role_key'),
      'apikey', current_setting('app.settings.anon_key')
    ),
    body := v_msg.message
  );
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

## Step 3: Cron Job

```sql
SELECT cron.schedule(
  'process-<type>-ai-queue',
  '* * * * *',  -- Every minute (adjust as needed)
  $$SELECT trigger_<type>_ai_queue_worker()$$
);
```

## Step 4: Edge Function

Create `supabase/functions/<type>-detector/index.ts`:

1. Parse queue message (msg_id, photo_id, tenant_id, storage_key)
2. Get signed URL for photo from Supabase Storage
3. Load detection type config via RPC (if configurable)
4. Build system prompt with detection type list
5. Call OpenAI Vision API with `gpt-4o` model
6. Apply results via `apply_<type>_detection_results` RPC
7. Delete message from queue on success
8. On failure: increment retry count or move to DLQ

## Step 5: Deploy

```bash
pnpm supabase functions deploy <type>-detector --project-ref <PROJECT_REF>
```

Set `verify_jwt = false` in `config.toml` for this function (invoked by cron, not by users).

## Realtime Broadcasts

AI detection processes broadcast status updates via Supabase Realtime:

### Channel Naming

```
# Multi-tenant
tenant:{tenantId}:<detection-type>

# Single-tenant
<detection-type>
```

### Event Payload

```typescript
{
  status: 'processing' | 'completed' | 'failed',
  photo_id: string,
  entity_id: string,       // The parent entity (order, item, etc.)
  duration_ms?: number,
  error?: string
}
```

### Broadcast Timing

- `processing` — immediately after dequeuing, before AI call
- `completed` — after successful processing and DB updates
- `failed` — when processing encounters an error

## Monitoring Queries

```sql
-- Queue status
SELECT COUNT(*) FILTER (WHERE vt <= NOW()) as available,
       COUNT(*) FILTER (WHERE vt > NOW()) as processing
FROM pgmq.q_<type>_ai_queue;

-- Recent runs
SELECT status, COUNT(*), AVG(duration_ms)
FROM <type>_ai_runs
WHERE created_at > NOW() - INTERVAL '24 hours'
GROUP BY status;

-- Dead letter queue
SELECT * FROM <type>_ai_dlq ORDER BY failed_at DESC LIMIT 10;
```
