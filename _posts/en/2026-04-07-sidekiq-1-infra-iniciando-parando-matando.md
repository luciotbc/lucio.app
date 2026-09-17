---
title: "Sidekiq in production: starting, stopping, and killing processes"
date: 2026-04-07 14:00:00 -0300
updated: 2026-04-07 14:00:00 -0300
tags: [ruby, rails, sidekiq, ops]
excerpt: "Shell commands to start, gracefully stop, and (as a last resort) forcefully kill a Sidekiq process. Why sidekiqctl stop should come first and when to reach for kill -9."
lang: en
ref: sidekiq-1-infra-iniciando-parando-matando
---

> **Sidekiq series** — part 1 of 4
> 1. **You are here — Infra: starting, stopping, killing**
> 2. [Diagnosis via console](/posts/sidekiq-2-diagnostico-pelo-console/)
> 3. [Bulk manipulation of jobs and processes](/posts/sidekiq-3-manipulacao-em-massa/)
> 4. [UI hacks in the dashboard](/posts/sidekiq-4-hacks-de-ui-no-painel/)

Before diving into queues and jobs, it's good to have the basics of the Sidekiq process lifecycle solid — because most incidents start with "how do I bring this down without losing jobs".

API documentation: [github.com/mperham/sidekiq/wiki/API](https://github.com/mperham/sidekiq/wiki/API).

## Starting Sidekiq

```bash
bundle exec sidekiq -d -L log/sidekiq.log -C config/sidekiq.yml
```

- `-d` daemonizes (releases the terminal).
- `-L log/sidekiq.log` points to the log file.
- `-C config/sidekiq.yml` points to the configuration file (queues, concurrency, retries).

In production this command usually lives in a systemd unit or the app's Procfile, but for local testing and VPS without an orchestrator it's perfect.

## Stopping via the Rails controller (recommended)

```bash
ps -ef | grep sidekiq | grep busy | grep -v grep | awk '{print $2}' > tmp/sidekiq.pid
cat tmp/sidekiq.pid
bundle exec sidekiqctl stop tmp/sidekiq.pid
```

`sidekiqctl stop` waits for running jobs to finish (up to the configured timeout) before killing the process. It's the polite way — preserves idempotency if a worker is in the middle of a non-atomic operation.

The pipe `ps -ef | grep busy` filters for the process that's actually working (not the master), and `awk '{print $2}'` extracts just the PID.

## Stopping via the OS (last resort)

```bash
ps -ef | grep sidekiq | grep busy | grep -v grep | awk '{print $2}'
kill -9 $(ps -ef | grep sidekiq | grep busy | grep -v grep | awk '{print $2}')
```

`kill -9` is the red button: the process dies immediately, with no chance to finish what it was doing. A running job becomes a retry, so only do this when:

- `sidekiqctl stop` is stuck and not responding
- The process has become a zombie consuming memory without doing anything
- You're in a fire situation where bringing it down is more important than preserving idempotency

If you find yourself using `kill -9` frequently, it's worth investigating why `stop` is taking so long — it's usually a job that doesn't respect the timeout or a stuck database connection.

## Next in the series

[Sidekiq diagnosis via console](/posts/sidekiq-2-diagnostico-pelo-console/) — `Sidekiq::Stats`, grouping by class, listing processes and workers.
