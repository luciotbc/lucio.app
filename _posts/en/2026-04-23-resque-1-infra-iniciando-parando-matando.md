---
title: "Resque in operations: starting, stopping, and killing workers"
date: 2026-04-23 14:00:00 -0300
updated: 2026-04-23 14:00:00 -0300
tags: [ruby, rails, resque, ops, redis]
excerpt: "How to start, gracefully drain, and (as a last resort) forcefully kill a Resque worker. The QUIT, TERM, USR1, USR2, and CONT signals — and when to use each one."
lang: en
ref: resque-1-infra-iniciando-parando-matando
---

> **Resque Series** — part 1 of 3
> 1. **You are here — Infra: starting, stopping, killing**
> 2. [Diagnosis via console](/posts/resque-2-diagnostico-pelo-console/)
> 3. [Mass manipulation of jobs and workers](/posts/resque-3-manipulacao-em-massa/)

> **Why this series exists** — I spent a long time working on projects that used Sidekiq, and during that process I compiled my notes into the [Sidekiq series](/posts/sidekiq-1-infra-iniciando-parando-matando/). I then ended up joining a project that uses Resque — that generated new notes, which I kept in the same structure as the ones I'd already written for Sidekiq. The operational problems are the same (queues that fill up, batch-failing jobs, deployments that need to drain workers), but the tools differ quite a bit. This series is the mirror of that one for topics that translate well: infra, diagnosis, and mass manipulation, with the equivalent commands in Resque. Where a topic has a direct parallel, I'll link to the corresponding post in the Sidekiq series.

The first important model difference: **Resque is process-per-job, Sidekiq is thread-per-job**. Each Resque worker listens to one or more queues and, when it picks up a job, `fork`s a child process that executes the job and exits. This changes how you start, drain, and kill — because there are always two related processes (the parent worker and the child that is executing).

API documentation: [github.com/resque/resque](https://github.com/resque/resque).

## Starting Resque

```bash
QUEUE=* bundle exec rake resque:work
```

- `QUEUE=*` makes the worker consume all queues that exist in Redis.
- `QUEUE=critical,high,default` listens in priority order — drains `critical` before touching `high`.

Starting multiple workers on a single machine:

```bash
COUNT=5 QUEUE=* bundle exec rake resque:workers
```

In the background with a PID file (useful for `kill` later):

```bash
PIDFILE=./tmp/pids/resque.pid BACKGROUND=yes QUEUE=* bundle exec rake resque:work
```

In production this lives in systemd or foreman, but for a VPS or test environment `BACKGROUND=yes + PIDFILE` covers the need.

> Unlike Sidekiq, there is no `-C config/sidekiq.yml`. Concurrency in Resque comes from starting N processes (not threads), so the "configuration" is how many `rake resque:work` processes you have running.

## Draining gracefully (QUIT — recommended)

```bash
kill -QUIT $(cat tmp/pids/resque.pid)
```

`QUIT` is the polite signal: the worker waits for the current job to finish (in the child process), then exits cleanly. The moral equivalent of `sidekiqctl stop` from [part 1 of the Sidekiq series](/posts/sidekiq-1-infra-iniciando-parando-matando/) — preserves idempotency if the worker is in the middle of a non-atomic operation.

If you don't have the PID file, you can extract it via `ps`:

```bash
ps -ef | grep '[r]esque' | grep -v 'master' | awk '{print $2}'
kill -QUIT $(ps -ef | grep '[r]esque' | grep -v 'master' | awk '{print $2}')
```

The `[r]esque` trick is the classic way to prevent `grep` itself from appearing in the results.

## Killing the child but keeping the worker (TERM/USR1)

Because Resque forks per job, you can kill **only the running job** without bringing down the parent worker:

```bash
# TERM on the worker: kills the child immediately AND shuts down the worker
kill -TERM $(cat tmp/pids/resque.pid)

# USR1 on the worker: kills the child immediately, but the worker stays up and picks the next job
kill -USR1 $(cat tmp/pids/resque.pid)
```

`USR1` is the right tool for "this specific job is stuck and holding the worker, but I don't want to take down the infrastructure". The job becomes a failure, the worker returns to the loop.

## Pausing without killing (USR2 / CONT)

```bash
# USR2: stops picking up new jobs (doesn't end the current one, just won't pick up the next)
kill -USR2 $(cat tmp/pids/resque.pid)

# CONT: resumes picking up jobs
kill -CONT $(cat tmp/pids/resque.pid)
```

This pair is Resque's equivalent of `Sidekiq::ProcessSet#quiet!` that appears in [part 3 of the Sidekiq series](/posts/sidekiq-3-manipulacao-em-massa/). Useful for deployment: `USR2` all workers, wait for in-progress jobs to finish, deploy, `CONT` to resume.

## Last resort (KILL)

```bash
kill -9 $(cat tmp/pids/resque.pid)
```

`KILL` (`-9`) kills the worker and orphans any child process in execution — those children become zombie processes until init reaps them, and the running job disappears without becoming a failure (because the worker had no time to register it). Use only when:

- `QUIT` is stuck and not responding
- The worker is consuming memory without doing anything visible
- There is a fire and bringing it down is more important than preserving state

## Signal summary

| Signal | Effect |
|--------|--------|
| `QUIT` | Finishes in-progress jobs, then exits (graceful) |
| `TERM` | Kills the child immediately and exits |
| `USR1` | Kills the child immediately, keeps the worker running |
| `USR2` | Stops picking up new jobs (pause) |
| `CONT` | Resumes picking up jobs (resume) |
| `KILL` (-9) | Kills the worker, orphans children |

## Next in the series

[Resque diagnosis via console](/posts/resque-2-diagnostico-pelo-console/) — `Resque.info`, queue counts, worker listing and what each one is doing.
