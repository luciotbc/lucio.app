---
title: "Resque diagnosis via console: info, counts, and workers"
date: 2026-04-24 14:00:00 -0300
updated: 2026-04-24 14:00:00 -0300
tags: [ruby, rails, resque, ops, redis]
excerpt: "Taking an X-ray of the Resque installation without touching anything: Resque.info, job counts per class in queues, the failure set, and listing what each worker is executing right now."
lang: en
ref: resque-2-diagnostico-pelo-console
---

> **Resque Series** — part 2 of 3
> 1. [Infra: starting, stopping, killing](/posts/resque-1-infra-iniciando-parando-matando/)
> 2. **You are here — Diagnosis via console**
> 3. [Mass manipulation of jobs and workers](/posts/resque-3-manipulacao-em-massa/)

> **Correlation with the Sidekiq series** — this post is the mirror of [part 2 of the Sidekiq series](/posts/sidekiq-2-diagnostico-pelo-console/). Same philosophy: before deleting or moving jobs, do a read-only pass to understand the state. The APIs are different (Resque exposes a lot through `Resque.info` and helper classes instead of `Sidekiq::Stats`/`Sidekiq::Queue`), but the questions you want to answer are the same: what is pending, what is failing, and who is working right now.

## General status

```ruby
Resque.info
# => {
#   :pending   => 1234,
#   :processed => 98765,
#   :queues    => 5,
#   :workers   => 8,
#   :working   => 3,
#   :failed    => 42,
#   :servers   => ["redis://localhost:6379/0"],
#   :environment => "production"
# }
```

In one call you get the full picture: pending (sum of all queues), total historically processed, number of queues, live workers, workers active *right now*, and total accumulated failures. It's the equivalent of `Sidekiq::Stats.new` from the previous series.

## Listing queues and their sizes

```ruby
Resque.queues
# => ["default", "critical", "mailers", "low"]

Resque.queues.map { |q| [q, Resque.size(q)] }.to_h
# => {"default"=>120, "critical"=>0, "mailers"=>3, "low"=>15000}
```

`Resque.size('queue_name')` is O(1) — it uses `LLEN` in Redis. You can call it freely.

## Total jobs in a queue by class

```ruby
queue_name = 'default'
Resque.peek(queue_name, 0, Resque.size(queue_name))
  .group_by { |job| job['class'] }
  .map { |k, v| [k, v.length] }
  .to_h
```

`Resque.peek(queue, start, count)` returns payloads without removing them (LRANGE in Redis). Since the payload is an already-parsed Hash, just group by `'class'`.

Useful for answering the same question as in the Sidekiq series: "which worker is dominating this queue?". In a post-deploy incident, it's usually a specific class producing jobs faster than the cluster can consume them.

> Caution: if the queue has millions of items, avoid loading everything. Take a sample (`Resque.peek(queue, 0, 5000)`) — that's usually enough to infer the distribution.

## Failures by class

Resque doesn't separate "retry" and "dead" like Sidekiq — every failure goes to the `failure backend` (usually Redis). Reading it is similar:

```ruby
total = Resque::Failure.count
Resque::Failure.all(0, total)
  .group_by { |f| f['payload']['class'] }
  .map { |k, v| [k, v.length] }
  .to_h
```

Shows which class is dominating the failure set. After a bad deploy, if a new class appears dominating the failures, it's a strong indicator of regression — same heuristic as in [part 2 of the Sidekiq series](/posts/sidekiq-2-diagnostico-pelo-console/).

## Failures by error message

One advantage of Resque: the failure stores the exception and message directly in the payload, so you can also group by error:

```ruby
Resque::Failure.all(0, Resque::Failure.count)
  .group_by { |f| "#{f['exception']}: #{f['error']}" }
  .map { |k, v| [k, v.length] }
  .sort_by { |_, v| -v }
  .first(10)
```

Top 10 most frequent errors. During an incident, this answers "are all 5,000 failures from the same `Net::OpenTimeout` or is there something new mixed in?" in seconds.

## Listing workers (all live processes)

```ruby
Resque.workers.each do |w|
  puts "#{w.to_s} | host=#{w.hostname} pid=#{w.pid} queues=#{w.queues.join(',')}"
end
```

Each worker registers itself in Redis when it starts and unregisters when it exits cleanly. If you see a worker listed but the process no longer exists (it crashed without QUIT), that's a *ghost worker* — I'll cover that in the next part.

## What each worker is executing right now

```ruby
Resque.working.each do |w|
  job = w.job
  puts "#{w.to_s} | class=#{job['payload'] && job['payload']['class']} queue=#{job['queue']} run_at=#{job['run_at']}"
end
```

`Resque.working` filters only those with a job in hand. Equivalent to `Sidekiq::Workers.new` from the previous series — shows threads/processes running **at this exact moment**, the class, and which queue.

When `Resque.info[:processed]` stopped going up but workers are still "working", this is where you find out who is stuck.

## How long each worker has been on a job

```ruby
require 'time'

Resque.working.each do |w|
  job = w.job
  next if job.empty?
  run_at = Time.parse(job['run_at'])
  puts "#{w.to_s} -> #{job['payload']['class']} running for #{(Time.now - run_at).to_i}s"
end
```

Finds stuck workers: if `running for` is in the thousands of seconds for a job that should take 200ms, someone got hung up on I/O.

## Ghost workers

```ruby
Resque.workers.reject { |w|
  hostname, pid, _ = w.id.split(':')
  hostname == `hostname`.strip && system("ps -p #{pid} > /dev/null 2>&1")
}
```

Lists workers registered in Redis whose process no longer exists on the machine where they run. In a cluster with multiple hosts, this check is only valid for local workers — for a real cluster, it's better to rely on `prune_dead_workers` (which I'll use in part 3).

## Next in the series

[Mass manipulation of jobs and workers](/posts/resque-3-manipulacao-em-massa/) — selecting failures by class, bulk requeue/remove, moving between queues, pausing workers via signal, and the nuclear option `Resque.redis.flushdb`.
