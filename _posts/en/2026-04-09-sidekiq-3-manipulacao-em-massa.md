---
title: "Sidekiq: bulk manipulation of jobs and processes"
date: 2026-04-09 14:00:00 -0300
updated: 2026-04-09 14:00:00 -0300
tags: [ruby, rails, sidekiq, ops, redis]
excerpt: "Snippets for selecting jobs by class, moving between queues, bulk retry or delete, quieting processes for deploy, and — when needed — wiping everything from Redis."
lang: en
ref: sidekiq-3-manipulacao-em-massa
---

> **Sidekiq series** — part 3 of 4
> 1. [Infra: starting, stopping, killing](/posts/sidekiq-1-infra-iniciando-parando-matando/)
> 2. [Diagnosis via console](/posts/sidekiq-2-diagnostico-pelo-console/)
> 3. **You are here — Bulk manipulation**
> 4. [UI hacks in the dashboard](/posts/sidekiq-4-hacks-de-ui-no-painel/)

After diagnosing (part 2), the invasive part usually follows: getting a problematic worker out of the way, reorganizing the queue, deleting jobs that no longer make sense. This post is the toolbox for that.

> The `; nil` at the end of blocks is just to prevent `rails console` from printing thousands of lines when the collection is large.

## Selecting retry jobs by class name

```ruby
job_class_name = 'SidekiqTest::SidekiqTestWorker'
jobs = Sidekiq::RetrySet.new.select { |job| job.klass == job_class_name }; nil
jobs.size
```

## Selecting dead jobs by class name

```ruby
job_class_name = 'SidekiqTest::SidekiqTestWorker'
jobs = Sidekiq::DeadSet.new.select { |job| job.klass == job_class_name }; nil
jobs.size
```

## Selecting queued jobs in a queue by class name

```ruby
queue_name = 'default'
job_class_name = 'SidekiqTest::SidekiqTestWorker'
jobs = Sidekiq::Queue.new(queue_name).select { |job| job.klass == job_class_name }; nil
jobs.count
```

## Re-queuing selected jobs for retry

```ruby
jobs.each(&:retry)
```

Applies to any collection of jobs from `RetrySet` or `DeadSet`. Moves everything back to the original queue.

## Deleting selected jobs

```ruby
jobs.each(&:delete)
```

Same pattern, with the inverse operation. Use once you're sure you can discard them — there's no going back.

## Moving 1000 jobs of a class to another queue

```ruby
queue_name = 'default'
new_queue_name = 'funnels_test_worker'
queue = Sidekiq::Queue.new(queue_name)

queue.first(1000).each do |job|
  if job.klass == "SidekiqTest::SidekiqTestWorker"
    SidekiqTest::SidekiqTestWorker.set(queue: new_queue_name).perform_async(*job.args)
    job.delete
  end
end; nil
```

This is the snippet that saves the day most often. When a worker is bringing down a shared queue, instead of pausing everything, I:

1. Create a dedicated queue (`funnels_test_worker`)
2. Move the problematic jobs to it
3. Spin up a separate Sidekiq process consuming that queue with reduced concurrency (e.g. 1 thread)

The rest of the operation doesn't feel it — and I can debug the worker calmly without pressuring the main queue.

## Quieting processes for deploy

```ruby
ps = Sidekiq::ProcessSet.new
ps.each(&:quiet!)
```

`quiet!` makes each process stop picking up new jobs, but finish the ones currently running. The polite way to drain before bringing down — good for deploys that don't use an orchestrator.

## Stopping processes via API

```ruby
ps = Sidekiq::ProcessSet.new
ps.each(&:stop!)
```

Equivalent to `sidekiqctl stop` from [part 1](/posts/sidekiq-1-infra-iniciando-parando-matando/), but via console instead of shell.

## Clear all of Sidekiq's Redis — DELETES EVERYTHING

```ruby
Sidekiq.redis { |conn| conn.flushdb }
```

Nuclear bomb: deletes **everything** Sidekiq has in Redis (queues, retries, dead set, stats). Only use this when you know you can reprocess safely or you're on a dev machine. In production, this is an incident — only do it with a clear recovery plan.

## Next in the series

[UI hacks for the Sidekiq dashboard](/posts/sidekiq-4-hacks-de-ui-no-painel/) — when the retries screen has 100,000 items and the native UI can't cope.
