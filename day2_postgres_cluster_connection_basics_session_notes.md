# PostgreSQL Cluster, Instance & Postmaster Basics — Session Notes

This document captures key concepts from the first technical/hands-on session, covering `initdb` and database cluster creation, `postgresql.conf` parameters, the cluster-vs-instance distinction, default databases, running multiple clusters/versions on one host, the postmaster's role, the connection-authentication-backend flow, and a live `kill -9` crash-recovery demo on the background writer. Each section includes terminal output reconstructed from the session, followed by all questions discussed.

---
## Table of Contents

- [1. Creating and Starting Your First Database Cluster](#1-creating-and-starting-your-first-database-cluster)
- [2. postgresql.conf — Enabling Connection Logging](#2-postgresqlconf--enabling-connection-logging)
- [3. Restart vs Reload](#3-restart-vs-reload)
- [4. Cluster vs Instance](#4-cluster-vs-instance)
- [5. Default Databases Created by initdb](#5-default-databases-created-by-initdb)
- [6. Running Multiple Clusters / Versions on One Host](#6-running-multiple-clusters--versions-on-one-host)
- [7. The Postmaster and Its Responsibilities](#7-the-postmaster-and-its-responsibilities)
- [8. Connection Flow — Authentication to Backend Process](#8-connection-flow--authentication-to-backend-process)
- [9. psql -h — Database User, Not OS User](#9-psql--h--database-user-not-os-user)
- [10. kill -9 on the Background Writer — Live Crash Recovery Demo](#10-kill--9-on-the-background-writer--live-crash-recovery-demo)
- [11. Summary Table — Terms Introduced This Session](#11-summary-table--terms-introduced-this-session)
- [12. Questions Discussed in This Session](#12-questions-discussed-in-this-session)

---

## 1. Creating and Starting Your First Database Cluster

```bash
$ cd /usr/pgsql-18/bin
$ ./initdb -D /u01/pgsql/18
```

`initdb` creates the first **database cluster** — in PostgreSQL terms, a cluster is simply a group of databases sharing one data directory, not a group of machines. It comes with **3 default databases** (see [Section 5](#5-default-databases-created-by-initdb)).

```bash
$ ./pg_ctl start -D /u01/pgsql/18
LOG:  database system is ready to accept connections
```

Confirm it's running — actual `ps -ef | grep postgres` output captured in session:

```
postgres    3618       1  0 07:46 ?        00:00:00 /usr/pgsql-18/bin/postgres -D /u01/pgsql/18
postgres    3619    3618  0 07:46 ?        00:00:00 postgres: logger
postgres    3620    3618  0 07:46 ?        00:00:00 postgres: io worker 0
postgres    3621    3618  0 07:46 ?        00:00:00 postgres: io worker 1
postgres    3622    3618  0 07:46 ?        00:00:00 postgres: io worker 2
postgres    3623    3618  0 07:46 ?        00:00:00 postgres: checkpointer
postgres    3624    3618  0 07:46 ?        00:00:00 postgres: background writer
postgres    3626    3618  0 07:46 ?        00:00:00 postgres: walwriter
postgres    3627    3618  0 07:46 ?        00:00:00 postgres: autovacuum launcher
postgres    3628    3618  0 07:46 ?        00:00:00 postgres: logical replication launcher
postgres    6373    3618  0 08:29 ?        00:00:00 postgres: postgres postgres [local] idle
postgres    6404    3618  0 08:29 ?        00:00:00 postgres: postgres postgres [local] idle
```

- **PID 3618** — the single long line containing `-D` — is the **postmaster**.
- Every process with PPID `3618` is a child of the postmaster: `logger`, `io worker 0/1/2`, `checkpointer`, `background writer`, `walwriter`, `autovacuum launcher`, `logical replication launcher` are all **background processes**; the two `postgres: postgres postgres [local] idle` lines (PID 6373, 6404) are **user backend processes** — one per connected client session.
- Note PG 18 introduces dedicated `io worker` processes (asynchronous I/O), which didn't exist as separate background processes in older versions.

---

## 2. postgresql.conf — Enabling Connection Logging

```bash
$ cd /u01/pgsql18
$ vi postgresql.conf
```

Three parameters were enabled during the demo:

```ini
log_connections    = on
log_disconnections = on
log_duration        = on
```

These are considered baseline/mandatory settings for any environment — without them, connect/disconnect events never show up in the server log.

Save, then restart to apply (see [Section 3](#3-restart-vs-reload)):

```bash
$ ./pg_ctl restart -D /u01/pgsql18
```

Tail the log after logging in and out once:

```bash
$ tail -f /u01/pgsql18/log/postgresql-*.log
LOG:  connection received: host=[local]
LOG:  connection authorized: user=postgres database=postgres
LOG:  connection authenticated: identity="postgres" method=peer
LOG:  disconnection: session time: 0:00:22.104 user=postgres database=postgres host=[local]
```

Without `log_connections` / `log_disconnections`, none of these four lines appear.

---

## 3. Restart vs Reload

Not every parameter needs the same treatment:

| Parameter | Change takes effect on |
|---|---|
| `log_connections`, `log_disconnections`, `log_duration` | restart (as demonstrated) |
| `max_connections` | **restart** (confirmed live — a `reload` alone did not pick up the new value) |

```bash
$ ./pg_ctl restart -D /u01/pgsql18
```

> Rule of thumb given in session: when unsure, try `pg_ctl reload`; if the new value doesn't show up in `SHOW <param>;`, it needs a restart.

---

## 4. Cluster vs Instance

This was the most repeated point of confusion in the session, resolved with a live count:

```bash
$ ps -ef | grep postgres | grep -c ' -D '
1
```

```bash
$ ls /u01 | grep pgsql
pgsql16   pgsql17   pgsql171   pgsql172   pgsql18   pgsql181
```

- **6 clusters exist on disk** (six `initdb`'d data directories).
- **Only 1 instance is running** (one line with `-D` in `ps -ef`).

| Term | Definition |
|---|---|
| Cluster | A group of databases under one data directory — created by `initdb`. Physical, on-disk. Not the same as an OS/HA cluster (group of machines). |
| Instance | The running postmaster + background processes serving one cluster, holding one dedicated slice of CPU/memory, bound to one port. Logical, in-memory/running. |

> Simplification used for the first few sessions only: "assume cluster and instance are the same for now." They are not — a cluster only becomes an instance once started, and a host can run several instances (several clusters, started, each on its own port) simultaneously.

---

## 5. Default Databases Created by initdb

```sql
postgres=# \l
                             List of databases
   Name    |  Owner   | Encoding |   Collate   |    Access privileges
-----------+----------+----------+-------------+-------------------------
 postgres  | postgres | UTF8     | en_US.UTF-8 |
 template0 | postgres | UTF8     | en_US.UTF-8 | =c/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | =c/postgres
```

| Database | Purpose | Mandatory? |
|---|---|---|
| `template0` | Pristine, unmodifiable seed database | Yes — always required |
| `template1` | Default template new databases are copied from; modifiable | No, but present by convention |
| `postgres` | Default working/admin database | No, but present by convention |

```sql
postgres=# CREATE DATABASE mx;
CREATE DATABASE
```

`CREATE DATABASE` copies its initial catalog/metadata from a template (`template1` by default) — it is not a parent-child or container/pluggable relationship, just a one-time copy at creation time.

```sql
postgres=# \c mx
mx=# \dt
```

All databases in a cluster share the **same** postmaster and the same CPU/memory allocation — there is no per-database resource isolation and no per-database listener.

---

## 6. Running Multiple Clusters / Versions on One Host

A second cluster, different port:

```bash
$ cd /usr/pgsql-18/bin
$ ./initdb -D /u01/pgsql181
$ ./pg_ctl start -D /u01/pgsql181 -o "-p 5433"
```

A third, on PostgreSQL 17 binaries entirely:

```bash
$ cd /usr/pgsql-17/bin
$ ./initdb -D /u01/pgsql17
$ ./pg_ctl start -D /u01/pgsql17 -o "-p 5434"
```

```bash
$ ps -ef | grep postgres | grep ' -D '
postgres  5651  1  /usr/pgsql-18/bin/postgres -D /u01/pgsql18    (port 5432)
postgres  6210  1  /usr/pgsql-18/bin/postgres -D /u01/pgsql181   (port 5433)
postgres  6390  1  /usr/pgsql-17/bin/postgres -D /u01/pgsql17    (port 5434)
```

- You can run **any number of instances** on one host — the only constraint is each instance needs its own port; running instances must never share a port.
- Different major versions (17 vs 18) can happily run side by side, but **cannot be grouped/merged** — each is a fully independent instance with its own binaries, data directory, postmaster, and background processes.

> ⚠️ Correction worth flagging: at one point in the session, database count per cluster was described as capped at "2 or 3 per machine." That is **not a real PostgreSQL limit** — a cluster can hold far more databases than that; the only real constraint is that they all share the instance's CPU/memory footprint, which is a capacity-planning concern, not a hard limit.

---

## 7. The Postmaster and Its Responsibilities

```
/usr/pgsql-18/bin/postgres -D /u01/pgsql/18   ← postmaster (PID 3618)
```

Three responsibilities called out explicitly in session:

1. **Connection establishment** — accepts incoming client connections and authenticates them (the "listener").
2. **Supervising background processes** — if a non-essential background process dies, the postmaster restarts it.
3. **Instance/crash recovery** — coordinates recovery via the startup process when the instance was not shut down cleanly.

> Oracle-background analogy given in session: postmaster ≈ Oracle's listener + PMON + SMON, combined into one process.

---

## 8. Connection Flow — Authentication to Backend Process

```
Client                     Postmaster                    Backend Process
  |  connection request →     |
  |                            | check pg_hba.conf rules
  |                            | ("the rulebook")
  |  ← authenticated           |
  |                            | fork() new backend process
  |  handed off to backend  →  |  ------------------------→  backend serves
  |                            |                              all queries for
  |                            |                              this session
```

1. Client contacts the postmaster, presenting connection info (host, database, user).
2. Postmaster evaluates this against its access-control rules (`pg_hba.conf`, referred to informally in session as "the rulebook") — e.g. allow/deny based on source address.
3. On success, the postmaster **forks a dedicated backend process** for that client.
4. From that point on, the client talks directly to its own backend process, not the postmaster, for the life of the session.

A second client connecting for different data (e.g. a `department` table query) goes through the identical three-step flow and gets its own, separate backend process.

There is no limit tied to a compiled constant for how many backends can be created — it's governed by the `max_connections` parameter (see [Section 3](#3-restart-vs-reload)).

---

## 9. psql -h — Database User, Not OS User

```bash
$ psql -h 192.168.1.147 -U postgres -d postgres
Password for user postgres:
```

Confirmed explicitly in Q&A: the credential prompted here is a **PostgreSQL database role's** password — entirely separate from OS-level (Linux) user accounts. Getting this wrong is a common early confusion for people coming from OS-prompt (`$`) vs `psql` prompt (`postgres=#`) contexts.

```bash
$ psql -h 192.168.1.147 -U postgres -d postgres
Password for user postgres:
psql: error: FATAL:  password authentication failed for user "postgres"
```

```
LOG:  connection received: host=192.168.1.147
LOG:  password authentication failed for user "postgres"
```

---

## 10. kill -9 on the Background Writer — Live Crash Recovery Demo

Background writer, PID 3624 (see the `ps -ef` output in [Section 1](#1-creating-and-starting-your-first-database-cluster)):

```bash
$ kill -9 3624
```

**Postmaster log response (actual capture):**

```
background writer process (PID 3624) was terminated by signal 9: Killed
terminating any other active server processes
all server processes terminated; reinitializing
database system was interrupted; last known up at 2026-07-02 08:21:05 IST
database system was not properly shut down; automatic recovery in progress
redo starts at 0/2E480E8
invalid record length at 0/2E481F0: expected at least 24, got 0
redo done at 0/2E481B8 system usage: CPU: user: 0.00 s, system: 0.00 s, elapsed: 0.00
```

> The `invalid record length ... expected at least 24, got 0` line is normal, not an error — it's simply WAL replay hitting the end of valid WAL records (the point up to which the last checkpoint had actually flushed), which is where redo correctly stops.

`ps -ef | grep postgres` immediately after recovery:

```
postgres    3618       1  0 07:46 ?        00:00:00 /usr/pgsql-18/bin/postgres -D /u01/pgsql/18
postgres    3619    3618  0 07:46 ?        00:00:00 postgres: logger
postgres    7597    3618  0 08:34 ?        00:00:00 postgres: io worker 0
postgres    7598    3618  0 08:34 ?        00:00:00 postgres: io worker 2
postgres    7599    3618  0 08:34 ?        00:00:00 postgres: io worker 1
postgres    7601    3618  0 08:34 ?        00:00:00 postgres: checkpointer
postgres    7602    3618  0 08:34 ?        00:00:00 postgres: background writer
postgres    7603    3618  0 08:34 ?        00:00:00 postgres: walwriter
postgres    7604    3618  0 08:34 ?        00:00:00 postgres: autovacuum launcher
postgres    7605    3618  0 08:34 ?        00:00:00 postgres: logical replication launcher
```

| Observation | Explanation |
|---|---|
| Postmaster PID unchanged (`3618`) | Postmaster is the parent driving recovery — killing a child never kills the postmaster; only killing the postmaster itself would leave nothing to auto-restart, requiring a manual `pg_ctl start` |
| `logger` PID unchanged (`3619`) | The logger is intentionally kept alive across a crash restart so that the recovery messages themselves have somewhere to be written — it is not torn down with the rest |
| Every other background process got a **new PID** (`3624 → 7602` for background writer, plus new PIDs for checkpointer, walwriter, autovacuum launcher, logical replication launcher, and all three io workers) | Postmaster terminated and re-forked the rest of the server processes as part of reinitializing the instance |
| The two user backend processes (PID 6373, 6404) no longer appear | Existing client sessions do **not** survive a crash restart — they are forcibly disconnected |

**What an active client session saw at the moment of the crash:**

```
postgres=# select *, pg_sleep(100) from emp;
WARNING:  terminating connection because of crash of another server process
DETAIL:  The postmaster has commanded this server process to roll back the current transaction
         and exit, because another server process exited abnormally and possibly corrupted
         shared memory.
HINT:  In a moment you should be able to reconnect to the database and repeat your command.
server closed the connection unexpectedly
        This probably means the server terminated abnormally
        before or while processing the request.
The connection to the server was lost. Attempting reset: Failed.
The connection to the server was lost. Attempting reset: Failed.
!?>
```

> Note the client did **not** auto-reconnect on the first two attempts (`Attempting reset: Failed` shown twice, ending in psql's disconnected `!?>` prompt) — `psql` retries the reset but only succeeds once the instance has actually finished recovery and is accepting connections again. If recovery is still in progress when the reset is attempted, the reset itself fails and a manual `\c` (or another attempt) is needed.

> Terminology correction made live in session: initially described as "restarted the server." After student pushback this was correctly refined to "restarted the **instance**" — the postmaster itself never died; it detected the abnormal exit of an essential child and drove crash recovery.

---

## 11. Summary Table — Terms Introduced This Session

| Term | Meaning |
|---|---|
| Cluster | Group of databases under one data directory, created by `initdb` |
| Instance | Running postmaster + background processes serving one cluster |
| Postmaster | Parent process: connection listener, background-process supervisor, recovery coordinator |
| Backend process | Per-client-connection process forked by the postmaster after authentication |
| `template0` | Unmodifiable seed database, always present |
| `template1` | Modifiable default template for new databases |
| `pg_hba.conf` ("the rulebook") | Access-control rules the postmaster checks before authenticating a connection |

---

## 12. Questions Discussed in This Session

**Q1. There's no "SQL prompt" like Oracle's — is everything OS-level here?**

No — `psql` is PostgreSQL's SQL client (the equivalent of Oracle's SQL*Plus). The `$` prompt is the OS shell; once inside `psql` the prompt changes (e.g. `postgres=#`). PostgreSQL genuinely blends OS-level operations (file copies for backup/recovery, log files on disk) with database-level operations more visibly than Oracle does, which is why it can feel "50% OS, 50% database" — but the SQL prompt is real and distinct from the shell.

---

**Q2. Is there a parent-child relationship between databases in a cluster, like Oracle CDB/PDB?**

No. All databases in a cluster share the same postmaster and the same CPU/memory allocation, but each database is otherwise independent — there's no ongoing parent-child link. `CREATE DATABASE` performs a one-time metadata copy from a template; it is not a pluggable/container relationship, and template0/template1 shouldn't be thought of as a literal CDB/PDB analogue beyond that first-pass mental model.

---

**Q3. Can a PostgreSQL 17 cluster and an 18 cluster be grouped together on the same host?**

No. Each major-version binary set drives its own fully independent instance. They can coexist on the same host on different ports, but they cannot be merged or grouped.

---

**Q4. Can I have separate listeners per database for security isolation?**

No. One postmaster serves every database in a cluster. Per-database access control is done via `pg_hba.conf` rules, not separate listeners.

---

**Q5. Cluster vs instance — is cluster physical and instance logical?**

Yes, confirmed live: 6 clusters existed on disk (`initdb`'d directories), but `ps -ef | grep postgres` showed only 1 running instance. Cluster = on-disk structure; instance = the running process tree serving a cluster.

---

**Q6. When `psql -h` prompts for a password, is that an OS user or a database user?**

Database user — a PostgreSQL role, unrelated to the OS-level Linux account. See [Section 9](#9-psql--h--database-user-not-os-user).

---

**Q7. Does `max_connections` need a restart or just a reload to take effect?**

Restart — confirmed live by attempting a reload first (value didn't change) and then a full restart (value applied).

---

**Q8. After the `kill -9` demo, why did the background writer's PID change but the postmaster's PID stay the same?**

Because the postmaster is the parent supervising the instance — it detected the abnormal termination of an essential child process (background writer), terminated the remaining backends/background processes, and drove crash recovery, restarting the instance. Only killing the postmaster itself would leave nothing to auto-restart, requiring a manual `pg_ctl start`. See [Section 10](#10-kill--9-on-the-background-writer--live-crash-recovery-demo).

Worth noting from the actual capture: the `logger` process (PID 3619) was the **one** background process that kept its original PID throughout — every other background process (checkpointer, walwriter, autovacuum launcher, logical replication launcher, io workers) got a new PID. That's because the logger needs to stay up to receive and write the recovery messages themselves; it isn't part of the tear-down/reinitialize cycle the way the other background processes are.
