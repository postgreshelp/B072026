# Connection Establishment — Day 2

Live-demo scripts for PgBouncer pooling behavior. Run in order; each builds on the last. All scripts connect through PgBouncer (`port=6432`), so `pg_stat_activity` shows Postgres-side state while `SHOW POOLS` / `SHOW CLIENTS` on the PgBouncer admin console shows the pooling side.

Pair each script with:
```sql
-- Postgres side (port 5432)
SELECT pid, usename, state, now() - query_start AS duration, query
FROM pg_stat_activity
ORDER BY pid;
```
```sql
-- PgBouncer admin console (psql -h host -p 6432 -U pgbouncer pgbouncer)
SHOW POOLS;
```

---

## 1. Connect and run `pg_sleep()` — baseline active session

Shows `state = active` for exactly 2 seconds, then the session disappears from `pg_stat_activity` on close.

```python
import psycopg2

conn = psycopg2.connect(
    host="192.168.108.147",
    port=6432,
    database="postgres",
    user="postgres",
    password="postgres"
)
cursor = conn.cursor()
cursor.execute("SELECT pg_sleep(2)")
cursor.close()
conn.close()
```

---

## 2. Make it idle in transaction

`psycopg2` defaults to `autocommit=False`, so `execute()` implicitly opens a transaction. Without a `commit()`/`rollback()` before the pause, the session sits as `idle in transaction` — the classic "app forgot to close its transaction" bug pattern.

```python
import psycopg2

conn = psycopg2.connect(
    host="192.168.108.147",
    port=6432,
    database="postgres",
    user="postgres",
    password="postgres"
)
cursor = conn.cursor()
cursor.execute("SELECT pg_sleep(2)")
input("Press Enter to continue...")
cursor.close()
conn.close()
```

---

## 3. Make it idle

Same as #2, but `conn.commit()` closes the transaction before the pause. Contrast with #2: `state = idle`, not `idle in transaction`.

```python
import psycopg2

conn = psycopg2.connect(
    host="192.168.108.147",
    port=6432,
    database="postgres",
    user="postgres",
    password="postgres"
)
cursor = conn.cursor()
cursor.execute("SELECT pg_sleep(2)")
conn.commit()
input("Press Enter to continue...")
cursor.close()
conn.close()
```

---

## 4. Simulate PgBouncer pooling — 40 connections vs 10 pool slots

40 concurrent sessions hit a pool sized for 10 (`default_pool_size = 10` in `pgbouncer.ini`). First 10 connect and run immediately; the rest queue inside PgBouncer until a slot frees up. Kill sessions from the database side using the cheat-sheet queries (`pg_terminate_backend`) — the `try/except` catches the resulting error cleanly instead of printing a raw traceback.

```python
import psycopg2
import threading

def run_session(session_id):
    try:
        conn = psycopg2.connect(
            host="192.168.108.147",
            port=6432,
            database="postgres",
            user="postgres",
            password="postgres"
        )
        cursor = conn.cursor()
        cursor.execute("SELECT pg_sleep(2)")
        conn.commit()
        cursor.close()
        conn.close()
    except psycopg2.OperationalError:
        print(f"Session {session_id}: terminated from PostgreSQL.")

threads = [threading.Thread(target=run_session, args=(i,)) for i in range(40)]
for t in threads:
    t.start()
```

> No `.join()` needed here — sessions are being killed externally via SQL, not tracked to completion by the script itself.

---

## 5. Simulate long-running queries

Same shape as #4, but a 60-second hold instead of 2 seconds — enough time to run the "find long-running queries" and "cancel/kill" queries from the cheat sheet against live sessions while they're still active. Note: with pool size 10 and 40 sessions at 60s each, the last batch won't finish for several minutes — pace the demo accordingly.

```python
import psycopg2
import threading

def run_session(session_id):
    try:
        conn = psycopg2.connect(
            host="192.168.108.147",
            port=6432,
            database="postgres",
            user="postgres",
            password="postgres"
        )
        cursor = conn.cursor()
        cursor.execute("SELECT pg_sleep(60)")
        conn.commit()
        cursor.close()
        conn.close()
    except psycopg2.OperationalError:
        print(f"Session {session_id}: terminated from PostgreSQL.")

threads = [threading.Thread(target=run_session, args=(i,)) for i in range(40)]
for t in threads:
    t.start()
```

---

## Suggested demo flow

1. Run **#1** — show a clean 2s active session in `pg_stat_activity`.
2. Run **#2** vs **#3** back to back — show `idle in transaction` vs `idle`, and why the distinction matters (idle-in-transaction holds locks).
3. Run **#4** — pool size 10, fire 40. Show `SHOW POOLS` (`cl_active=10, cl_waiting=30`) next to `pg_stat_activity` (only 10 rows). This is the core "PgBouncer absorbs the overflow" moment.
4. Run **#5** — same setup, longer hold, so there's time to run the cheat-sheet queries (`pg_cancel_backend`, `pg_terminate_backend`) live against real sessions and watch `cl_waiting` drop in real time as slots free up.
