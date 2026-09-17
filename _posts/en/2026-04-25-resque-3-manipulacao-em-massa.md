---
title: "Resque: mass manipulation of jobs and workers"
date: 2026-04-25 14:00:00 -0300
updated: 2026-04-25 14:00:00 -0300
tags: [ruby, rails, resque, ops, redis]
excerpt: "Snippets for selecting failures by class, bulk requeue/remove, moving jobs between queues, pausing workers via USR2/CONT, pruning ghost workers, and — when needed — clearing everything from Redis."
lang: en
ref: resque-3-manipulacao-em-massa
---

> **Resque Series** — part 3 of 3
> 1. [Infra: starting, stopping, killing](/posts/resque-1-infra-iniciando-parando-matando/)
> 2. [Diagnosis via console](/posts/resque-2-diagnostico-pelo-console/)
> 3. **You are here — Mass manipulation**

> **Correlation with the Sidekiq series** — this post is the mirror of [part 3 of the Sidekiq series](/posts/sidekiq-3-manipulacao-em-massa/). The big difference is that Resque doesn't have `RetrySet`/`DeadSet` — everything that fails goes to `Resque::Failure`, indexed by position (not by ID). This changes how to do "bulk delete": since removing an item shifts the indexes, you need to iterate backwards. This care will appear in almost every snippet below.

## Selecting failures by class

```ruby
class_name = 'MyApp::ImportWorker'
total = Resque::Failure.count
failures = Resque::Failure.all(0, total).each_with_index.select { |f, _| f['payload']['class'] == class_name }
failures.size
```

Unlike Sidekiq (`RetrySet#select { |j| j.klass == ... }`), the Resque collection here is a linear indexed list. I load it with `each_with_index` to preserve the original position — I'll need it for retry/delete.

## Selecting failures by error message

```ruby
needle = 'Net::OpenTimeout'
failures = Resque::Failure.all(0, Resque::Failure.count).each_with_index.select { |f, _|
  f['error'].to_s.include?(needle)
}
failures.size
```

Works well combined with the count by error from the previous post: you discover that 90% of failures are from a single `OpenTimeout`, isolate only those, and retry after the external service comes back.

## Requeueing an individual failure

```ruby
Resque::Failure.requeue(0)  # index 0 is the oldest failure
```

`requeue` puts the job back in the original queue without removing it from the failure set — so you can still see the history afterwards.

## Removing an individual failure

```ruby
Resque::Failure.remove(0)
```

Removes only that entry from the failure set. **Important:** all failures after it shift their index by 1.

## Bulk retry for a class — iterating backwards

```ruby
class_name = 'MyApp::ImportWorker'
indexes = Resque::Failure.all(0, Resque::Failure.count)
  .each_with_index
  .select { |f, _| f['payload']['class'] == class_name }
  .map { |_, i| i }

indexes.reverse_each do |i|
  Resque::Failure.requeue(i)
  Resque::Failure.remove(i)
end
```

This pattern is specific to Resque: since `remove` shifts indexes, **you must iterate from largest to smallest** so each operation doesn't invalidate the previous ones. If you do `indexes.each` (ascending order), you'll delete/reschedule the wrong job from the second iteration onwards.

> `Resque::Failure.requeue_all` and `Resque::Failure.clear` exist for acting on all failures without a filter. Use them when the filter is "everything".

## Deleting all failures

```ruby
Resque::Failure.clear
```

Equivalent to `DeadSet#clear`. No going back — only do this after confirming via diagnosis (part 2) that there's nothing useful there.

## Moving jobs from a class to a dedicated queue

```ruby
queue_name     = 'default'
new_queue_name = 'isolated_import'
class_name     = 'MyApp::ImportWorker'

snapshot = Resque.peek(queue_name, 0, Resque.size(queue_name))
moved = 0

snapshot.each_with_index do |payload, _|
  next unless payload['class'] == class_name
  # removes the first exact occurrence of this payload from the queue
  removed = Resque.redis.lrem("queue:#{queue_name}", 1, Resque.encode(payload))
  if removed > 0
    Resque.push(new_queue_name, payload)
    moved += 1
  end
end

moved
```

Direct equivalent of the "Moving 1000 jobs from one class to another queue" snippet from the [Sidekiq series](/posts/sidekiq-3-manipulacao-em-massa/). The mechanics differ because Resque stores jobs as a `LIST` in Redis (`queue:<name>`), so using `LREM` directly is the reliable way to remove by exact payload without touching neighbors.

Why isolate: if a class is overwhelming a shared queue, instead of pausing everything, I create a dedicated queue (`isolated_import`), move the problematic jobs there, and spin up a separate worker consuming only that queue. The rest of the operation doesn't feel it.

```bash
QUEUE=isolated_import COUNT=1 bundle exec rake resque:workers
```

## Deleting an entire queue

```ruby
Resque.remove_queue('isolated_import')
```

Deletes both the content (`queue:<name>`) and the queue's registration in the queue index. Useful for a temporarily created queue like above — without this, it keeps showing up in `Resque.queues` even when empty.

## Pausing all workers (USR2)

```ruby
Resque.workers.each do |w|
  hostname, pid, _ = w.id.split(':')
  next unless hostname == Socket.gethostname  # only local workers
  Process.kill('USR2', pid.to_i) rescue nil
end
```

Since Resque doesn't have a "via Redis" API to pause a worker (unlike `Sidekiq::Process#quiet!`), the only way is to send a signal to the process. That's why the hostname filter: you can only signal workers running on the same machine where you're running the console.

To resume:

```ruby
Resque.workers.each do |w|
  hostname, pid, _ = w.id.split(':')
  next unless hostname == Socket.gethostname
  Process.kill('CONT', pid.to_i) rescue nil
end
```

> In a multi-host cluster, the alternative is to run these snippets via Capistrano/Ansible on each node, or orchestrate with systemd. "Via Redis" pause doesn't exist.

## Pruning ghost workers

```ruby
Resque.workers.each(&:prune_dead_workers)
```

Workers that died without calling `unregister_worker` (e.g., `kill -9` or crash) remain listed in Redis but without a corresponding process. `prune_dead_workers` checks each worker on the current host and unregisters those that have no live process. Run on each host periodically — in production, I keep it in a cron or in the health check of the systemd unit.

## Clearing all of Resque's Redis — DELETES EVERYTHING

```ruby
Resque.redis.flushdb
```

Nuclear option, exact equivalent of `Sidekiq.redis { |conn| conn.flushdb }` from the previous series: deletes **everything** Resque has in Redis (queues, registered workers, failures, stats). Use only when you know you can reprocess safely, or in development. In production, this is an incident — only do it with a clear recovery plan.

> If Redis has other uses in the same db (cache, sessions, other workers), `flushdb` deletes **all of that too**. Check `Resque.redis.client.db` first — keeping Sidekiq/Resque/cache in different DBs (`/0`, `/1`, `/2`) is the standard precisely for this scenario.

## End of the series

This was part 3 and the last one. Going back to the beginning: [Infra: starting, stopping, killing](/posts/resque-1-infra-iniciando-parando-matando/).

And if you're administering both at the same time (classic case: legacy service on Resque + new services on Sidekiq), it's worth having both series side by side. The [first part of the Sidekiq series](/posts/sidekiq-1-infra-iniciando-parando-matando/) is the equivalent entry point.
