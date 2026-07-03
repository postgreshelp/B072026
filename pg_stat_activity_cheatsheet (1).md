# pg_stat_activity — Session Management Cheat Sheet

Quick reference for inspecting and managing PostgreSQL client sessions. Useful for live demos, troubleshooting, and classroom sessions on connection pooling.

---

## 1. Long-running active queries

```sql
SELECT pid,
       usename,
       application_name,
       client_addr,
       state,
       now() - query_start AS duration,
       query
FROM pg_stat_activity
WHERE state = 'active'
  AND pid <> pg_backend_pid()
ORDER BY duration DESC;
```

## 2. Idle sessions

```sql
SELECT pid,
       usename,
       application_name,
       client_addr,
       state,
       now() - state_change AS idle_time,
       query
FROM pg_stat_activity
WHERE state = 'idle'
ORDER BY idle_time DESC;
```

## 3. Idle-in-transaction sessions

```sql
SELECT pid,
       usename,
       application_name,
       client_addr,
       state,
       now() - state_change AS idle_time,
       query
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY idle_time DESC;
```

## 4. Cancel all active long-running queries

Cancels only the current query, not the connection.

```sql
SELECT pg_cancel_backend(pid)
FROM pg_stat_activity
WHERE state = 'active'
  AND pid <> pg_backend_pid();
```

## 5. Kill all idle sessions

Disconnects every idle session except your own.

```sql
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle'
  AND pid <> pg_backend_pid();
```

## 6. Kill all idle-in-transaction sessions

```sql
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle in transaction'
  AND pid <> pg_backend_pid();
```

## 7. Kill every client connection except yourself

```sql
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE pid <> pg_backend_pid();
```

> ⚠️ Use this only in a lab environment.

## 8. Kill sessions idle for more than 5 minutes

```sql
SELECT pg_terminate_backend(pid)
FROM pg_stat_activity
WHERE state = 'idle'
  AND now() - state_change > interval '5 minutes'
  AND pid <> pg_backend_pid();
```

## 9. Cancel queries running for more than 30 seconds

```sql
SELECT pg_cancel_backend(pid)
FROM pg_stat_activity
WHERE state = 'active'
  AND now() - query_start > interval '30 seconds'
  AND pid <> pg_backend_pid();
```

## 10. Show all sessions (best classroom query)

```sql
SELECT pid,
       usename,
       application_name,
       client_addr,
       backend_start,
       state,
       wait_event_type,
       wait_event,
       now() - query_start AS query_duration,
       now() - state_change AS state_duration,
       query
FROM pg_stat_activity
ORDER BY pid;
```

---

## Bonus: PgBouncer admin console (the missing half)

`pg_stat_activity` only shows connections that have actually reached Postgres. It has **no visibility into clients queued inside PgBouncer** waiting for a free slot. To see that side, connect to PgBouncer's special admin database:

```bash
psql -h localhost -p 6432 -U pgbouncer pgbouncer
```

```sql
SHOW POOLS;     -- cl_active, cl_waiting, sv_active, sv_idle per database/user pair
SHOW CLIENTS;   -- one row per client connection, state = active/waiting
SHOW SERVERS;   -- one row per real backend connection PgBouncer is holding open
```

**Pairing for a live demo:** run query #10 against Postgres directly (port 5432) in one pane, and `SHOW POOLS` against PgBouncer's admin console in another. When the pool is saturated you'll see something like `cl_active=10, cl_waiting=10` on the PgBouncer side while Postgres only ever shows 10 rows — the exact proof that PgBouncer absorbs the overflow before it ever reaches the database.
