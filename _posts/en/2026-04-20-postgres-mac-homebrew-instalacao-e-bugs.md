---
title: "PostgreSQL on Mac with Homebrew: installation and common bugs"
date: 2026-04-20 14:00:00 -0300
updated: 2026-04-20 14:00:00 -0300
tags: [postgresql, mac, homebrew, setup, ops]
excerpt: "Getting Postgres running on Mac via brew is easy — until the first time it won't start. Commands to install, read the log, remove a lock file, and migrate the data directory after a major upgrade."
lang: en
ref: postgres-mac-homebrew-instalacao-e-bugs
---

> **PostgreSQL series** — part 1 of 2
>
> 1. **You are here — Installation on Mac and common bugs**
> 2. [Tuning and investigation via the SQL console](/posts/postgres-tuning-e-investigacao-pelo-console/)

Running Postgres on Mac via Homebrew is trivial when everything is fine. The problem is that when something gets stuck, the error message rarely tells you what to do. This post is the sequence of commands I run, in order, when `brew services start` fails — plus what I do after a major version upgrade.

## Installing

```bash
brew install postgresql@14
```

I always pin the version (`@14`, `@15`, `@16`) instead of using the unversioned formula. Postgres major versions change the on-disk data format, and installing "the latest" without noticing is a recipe for losing your database.

## Starting, stopping, restarting

```bash
brew services start postgresql@14
brew services stop postgresql@14
brew services restart postgresql@14
```

Straightforward. The service keeps running between reboots; to run Postgres just "right now" use `pg_ctl -D /opt/homebrew/var/postgresql@14 start` directly.

## When it won't start: read the log

If `brew services start` returns `error` or `psql` refuses connections, before anything else look at the log:

```bash
brew services stop postgresql@14
rm /opt/homebrew/var/log/postgresql@14.log
brew services start postgresql@14
cat /opt/homebrew/var/log/postgresql@14.log
```

Deleting the log before starting clears the noise from previous sessions — you only read what happened in this attempt.

## Error: "lock file postmaster.pid already exists"

```text
FATAL:  lock file "postmaster.pid" already exists
```

This happens when Postgres died without shutting down properly (kernel panic, `kill -9`, dead battery). The process is no longer there, but the lock file stayed behind. Solution:

```bash
rm -rf /opt/homebrew/var/postgresql@14/postmaster.pid
brew services start postgresql@14
```

If you're not sure that no Postgres process is running, check first with `ps -ef | grep postgres | grep -v grep`. Deleting the lock while the process is alive can cause corruption.

## After a `brew upgrade`: data directory moved

When you upgrade Postgres by a major version, Homebrew changes the data directory. The log makes it clear:

```text
You can migrate to a versioned data directory by running:
  mv -v "/opt/homebrew/var/postgres" "/opt/homebrew/var/postgresql@14"
```

Before moving, **stop the service**:

```bash
brew services stop postgresql@14
mv -v /opt/homebrew/var/postgres /opt/homebrew/var/postgresql@14
brew services start postgresql@14
```

If you already started the service without migrating and it created an empty cluster at `postgresql@14`, you can end up with two data dirs — a new empty one and another with your data. I've lost a database this way. If that happens: stop the service, delete the empty cluster, and redo the `mv` before starting again.

To see details of the installed formula:

```bash
brew info postgresql@14
```

## Creating default users

A fresh Postgres install on Mac doesn't have the `postgres` user (by default the owner is your system user). To have what other tools expect:

```bash
createuser -s postgres
createuser --interactive --pwprompt
createdb init_test
```

- `createuser -s postgres` creates the `postgres` superuser (without password). Useful for tools that assume this user.
- `createuser --interactive --pwprompt` runs a wizard to create a regular user with a password.
- `createdb init_test` creates a test database just to confirm everything is working.

## Enabling `pg_stat_statements`

`pg_stat_statements` is the most useful extension for anyone who wants to understand slow queries. It comes bundled with the installation, but needs to be loaded at boot.

Edit `/opt/homebrew/var/postgresql@14/postgresql.conf` and add:

```ini
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.track = all
```

Or from the command line:

```bash
echo "shared_preload_libraries = 'pg_stat_statements'" >> /opt/homebrew/var/postgresql@14/postgresql.conf
echo "pg_stat_statements.track = all" >> /opt/homebrew/var/postgresql@14/postgresql.conf
brew services restart postgresql@14
```

Restart the service (a reload isn't enough — `shared_preload_libraries` is only read at startup) and create the extension in a database where you want to measure:

```sql
CREATE EXTENSION pg_stat_statements;
SELECT * FROM pg_stat_statements LIMIT 5;
```

If the query above returns rows, it's working. How to use this to hunt down slow queries is left for the [next post in the series](/posts/postgres-tuning-e-investigacao-pelo-console/).

Reference for enabling it in more depth: [bytebase.com/docs/slow-query/enable-pg-stat-statements-for-postgresql](https://www.bytebase.com/docs/slow-query/enable-pg-stat-statements-for-postgresql/).

## Next in the series

[Tuning and investigation via the SQL console](/posts/postgres-tuning-e-investigacao-pelo-console/) — `EXPLAIN ANALYZE` in PEV2, adjusting `work_mem`, listing index usage, counting rows across all tables and more.
