---
title: "Sidekiq diagnosis via console: stats, counts, and processes"
date: 2026-04-08 14:00:00 -0300
updated: 2026-04-08 14:00:00 -0300
tags: [ruby, rails, sidekiq, ops, redis]
excerpt: "How to take an X-ray of your Sidekiq installation without touching anything: Sidekiq::Stats, grouping jobs by class, and listing processes and workers."
lang: en
ref: sidekiq-2-diagnostico-pelo-console
---

> **Sidekiq series** — part 2 of 4
> 1. [Infra: starting, stopping, killing](/posts/sidekiq-1-infra-iniciando-parando-matando/)
> 2. **You are here — Diagnosis via console**
> 3. [Bulk manipulation of jobs and processes](/posts/sidekiq-3-manipulacao-em-massa/)
> 4. [UI hacks in the dashboard](/posts/sidekiq-4-hacks-de-ui-no-painel/)

When something seems off, before deleting or moving jobs, I always do a "read" pass — understand the state before touching anything. This post gathers the read-only commands I use for that step.

## General status

```ruby
stats = Sidekiq::Stats.new
stats.processed
stats.failed
stats.scheduled_size
stats.retry_size
stats.dead_size
stats.processes_size
stats.default_queue_latency
stats.workers_size
stats.enqueued
```

In 5 seconds you know: how many jobs have been processed, how many failed, what's the size of the retry/dead set, what's the default queue latency. It's a mental dashboard without needing the web panel.

## Total jobs in a queue by class

```ruby
queue_name = 'default'
Sidekiq::Queue.new(queue_name).map { |j| j.klass }.group_by { |e| e }.map { |k, v| [k, v.length] }.to_h
```

Useful for answering "which worker is dominating this queue?". When the queue fills up, it's usually a specific class that's producing jobs faster than the cluster can consume them.

## Total retry jobs by class

```ruby
Sidekiq::RetrySet.new.map { |j| j.klass }.group_by { |e| e }.map { |k, v| [k, v.length] }.to_h
```

Shows which worker is failing the most. Useful after a deploy: if a new class appears dominating the retry set, it's a strong indicator of a regression.

## Total dead jobs by class

```ruby
Sidekiq::DeadSet.new.map { |j| j.klass }.group_by { |e| e }.map { |k, v| [k, v.length] }.to_h
```

The dead set is the graveyard: jobs that exhausted all retries. If you see numbers growing here that weren't expected, it's worth investigating — you're literally losing work.

## Listing running processes

```ruby
ps = Sidekiq::ProcessSet.new
ps.size
ps.each do |process|
  p "pid: #{process['pid']} hostname: #{process['hostname']} busy: #{process['busy']} quiet: #{process['quiet']} queues: #{process['queues'].length} [#{process['queues'].sort.join(', ')}]"
end
```

Shows each live Sidekiq process in the cluster with:
- `busy` — how many threads are working right now
- `quiet` — whether it was paused (not picking up new jobs)
- `queues` — which queues this process consumes

In a multi-node cluster, this helps you understand if one of the processes died without notice.

## Inspecting workers (active threads right now)

```ruby
workers = Sidekiq::Workers.new
workers.size
workers.each do |process_id, thread_id, work|
  p "process_id: #{process_id} thread_id: #{thread_id} queue: #{work['queue']} retry: #{work['payload']['retry']} class: #{work['payload']['class']}"
end
```

`Workers` is different from `ProcessSet`: here you see **threads executing at this exact moment** — what class each one is running, in which queue, whether it's a retry attempt. Helps you understand what's stalling when the processed counter stopped rising.

## Next in the series

[Bulk manipulation of jobs and processes](/posts/sidekiq-3-manipulacao-em-massa/) — selecting jobs by class, moving between queues, bulk retry/delete, quieting processes for deploy.
