---
name: supabase-postgres-best-practices
description: Postgres performance optimization and best practices for Supabase projects. Use when writing or reviewing SQL queries, designing schemas, implementing indexes, configuring connection pooling, or optimizing Row-Level Security (RLS) performance. Covers 8 categories from critical query performance to advanced features.
---

# Supabase Postgres Best Practices

Performance optimization guide for Postgres, based on Supabase's official recommendations. Rules are organized by priority to guide schema design and query optimization.

## When to Apply

Reference these guidelines when:
- Writing SQL queries or designing schemas
- Implementing indexes or query optimization
- Reviewing database performance issues
- Configuring connection pooling or scaling
- Optimizing Row-Level Security (RLS)

## Rule Categories by Priority

| Priority | Category | Impact | Key Rules |
|----------|----------|--------|-----------|
| 1 | Query Performance | CRITICAL | Missing indexes, composite indexes, covering indexes, partial indexes, index types |
| 2 | Connection Management | CRITICAL | Connection pooling, prepared statements, idle timeout, connection limits |
| 3 | Security & RLS | CRITICAL | RLS basics, RLS performance, privilege management |
| 4 | Schema Design | HIGH | Data types, foreign key indexes, lowercase identifiers, partitioning, primary keys |
| 5 | Concurrency & Locking | MEDIUM-HIGH | Advisory locks, deadlock prevention, short transactions, SKIP LOCKED |
| 6 | Data Access Patterns | MEDIUM | Batch inserts, N+1 queries, pagination, upsert |
| 7 | Monitoring & Diagnostics | LOW-MEDIUM | EXPLAIN ANALYZE, pg_stat_statements, VACUUM/ANALYZE |
| 8 | Advanced Features | LOW | Full-text search, JSONB indexing |

## Critical Rules (Inline)

### 1. Missing Indexes

**Every column used in WHERE, JOIN, or ORDER BY should have an index.**

```sql
-- Check for missing indexes
SELECT relname, seq_scan, idx_scan,
       CASE WHEN seq_scan > 0
            THEN round(100.0 * idx_scan / (seq_scan + idx_scan), 1)
            ELSE 100 END AS idx_scan_pct
FROM pg_stat_user_tables
WHERE seq_scan > 1000
ORDER BY seq_scan DESC;
```

```sql
-- Add indexes for common query patterns
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_created_at ON orders(created_at DESC);
CREATE INDEX idx_orders_tenant_id ON orders(tenant_id);  -- Critical for multi-tenant RLS
```

### 2. RLS Performance

**Use `(select auth.uid())` instead of bare `auth.uid()` in policies — the subquery form caches per-statement.**

```sql
-- SLOW: auth.uid() evaluated per row
CREATE POLICY "bad" ON orders FOR SELECT
  USING (user_id = auth.uid());

-- FAST: subquery evaluated once per statement
CREATE POLICY "good" ON orders FOR SELECT
  USING (user_id = (select auth.uid()));
```

**Index the columns used in RLS policies:**

```sql
-- For single-tenant: index user_id
CREATE INDEX idx_orders_user_id ON orders(user_id);

-- For multi-tenant: index tenant_id (used in policy lookups)
CREATE INDEX idx_orders_tenant_id ON orders(tenant_id);
CREATE INDEX idx_user_tenant_roles_user_id ON user_tenant_roles(user_id);
```

### 3. Connection Pooling

Always use Supabase's built-in connection pooler (pgBouncer on port 6543) for application connections. Direct connections (port 5432) should only be used for migrations and admin tasks.

```
# Application connection string (pooled)
postgresql://postgres:[password]@db.[ref].supabase.co:6543/postgres

# Migration/admin connection string (direct)
postgresql://postgres:[password]@db.[ref].supabase.co:5432/postgres
```

### 4. Batch Inserts

Use bulk inserts instead of row-by-row:

```typescript
// SLOW: N individual inserts
for (const item of items) {
   await supabase.from('items').insert(item);
}

// FAST: Single batch insert
await supabase.from('items').insert(items);
```

### 5. Pagination

Use cursor-based pagination for large datasets:

```typescript
// SLOW: OFFSET-based (scans all skipped rows)
const { data } = await supabase
   .from('items')
   .select('*')
   .range(10000, 10049);

// FAST: Cursor-based (seeks directly)
const { data } = await supabase
   .from('items')
   .select('*')
   .gt('id', lastSeenId)
   .order('id')
   .limit(50);
```

### 6. Foreign Key Indexes

**Every foreign key column needs an index.** Without one, cascading deletes and joins trigger sequential scans.

```sql
-- Common pattern: add index for every FK
CREATE INDEX idx_order_items_order_id ON order_items(order_id);
CREATE INDEX idx_photos_entity_id ON photos(entity_id);
```

### 7. N+1 Query Prevention

Use Supabase's built-in relation queries instead of manual joins:

```typescript
// BAD: N+1 queries
const orders = await supabase.from('orders').select('*');
for (const order of orders.data) {
   const items = await supabase.from('items').select('*').eq('order_id', order.id);
}

// GOOD: Single query with embedded relation
const { data } = await supabase
   .from('orders')
   .select('*, items(*)');
```

## Additional Patterns

### EXPLAIN ANALYZE

Always use `EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)` to profile slow queries. Look for:
- `Seq Scan` on large tables (needs index)
- `Sort` without index (add covering index)
- High `Buffers: shared read` (table needs VACUUM or is too large)

### VACUUM and ANALYZE

Supabase runs autovacuum, but after bulk operations run manually:

```sql
VACUUM ANALYZE tablename;
```

### Upsert Pattern

```sql
INSERT INTO items (id, name, updated_at)
VALUES ('123', 'Widget', now())
ON CONFLICT (id) DO UPDATE SET
  name = EXCLUDED.name,
  updated_at = EXCLUDED.updated_at;
```

## References

- https://www.postgresql.org/docs/current/
- https://supabase.com/docs/guides/database/overview
- https://supabase.com/docs/guides/auth/row-level-security
- https://supabase.com/docs/guides/database/extensions/pg_stat_statements
