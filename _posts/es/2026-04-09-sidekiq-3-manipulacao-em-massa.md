---
title: "Sidekiq: manipulación masiva de jobs y procesos"
date: 2026-04-09 14:00:00 -0300
updated: 2026-04-09 14:00:00 -0300
tags: [ruby, rails, sidekiq, ops, redis]
excerpt: "Snippets para seleccionar jobs por clase, moverlos entre queues, hacer retry o delete masivo, pausar procesos para deploy y — cuando es necesario — limpiar todo de Redis."
lang: es
ref: sidekiq-3-manipulacao-em-massa
---

> **Serie Sidekiq** — parte 3 de 4
> 1. [Infra: iniciando, deteniendo, matando](/posts/sidekiq-1-infra-iniciando-parando-matando/)
> 2. [Diagnóstico por consola](/posts/sidekiq-2-diagnostico-pelo-console/)
> 3. **Estás aquí — Manipulación masiva**
> 4. [Hacks de UI en el panel](/posts/sidekiq-4-hacks-de-ui-no-painel/)

Después de diagnosticar (parte 2), generalmente viene la parte invasiva: sacar un worker problemático del camino, reorganizar la queue, borrar jobs que ya no tienen sentido. Este post es la caja de herramientas para eso.

> El `; nil` al final de los bloques es solo para evitar que `rails console` imprima miles de líneas cuando la colección es grande.

## Seleccionando jobs en retry por nombre de clase

```ruby
job_class_name = 'SidekiqTest::SidekiqTestWorker'
jobs = Sidekiq::RetrySet.new.select { |job| job.klass == job_class_name }; nil
jobs.size
```

## Seleccionando jobs muertos por nombre de clase

```ruby
job_class_name = 'SidekiqTest::SidekiqTestWorker'
jobs = Sidekiq::DeadSet.new.select { |job| job.klass == job_class_name }; nil
jobs.size
```

## Seleccionando jobs encolados en una queue por nombre de clase

```ruby
queue_name = 'default'
job_class_name = 'SidekiqTest::SidekiqTestWorker'
jobs = Sidekiq::Queue.new(queue_name).select { |job| job.klass == job_class_name }; nil
jobs.count
```

## Reencolando jobs seleccionados para retry

```ruby
jobs.each(&:retry)
```

Aplica a cualquier colección de jobs proveniente de `RetrySet` o `DeadSet`. Mueve todo de vuelta a la queue de origen.

## Borrando jobs seleccionados

```ruby
jobs.each(&:delete)
```

Mismo patrón, con la operación inversa. Úsalo cuando estés seguro de que podés descartar — no hay vuelta atrás.

## Moviendo 1000 jobs de una clase a otra queue

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

Este es el snippet que más salva el día. Cuando un worker está hundiendo una queue compartida, en vez de pausar todo, yo:

1. Creo una queue dedicada (`funnels_test_worker`)
2. Muevo los jobs problemáticos a ella
3. Levanto un proceso Sidekiq separado consumiendo esa queue con concurrencia reducida (ej: 1 thread)

El resto de la operación no lo nota — y puedo debuguear el worker con calma sin presionar la queue principal.

## Pausando procesos para deploy (quiet)

```ruby
ps = Sidekiq::ProcessSet.new
ps.each(&:quiet!)
```

`quiet!` hace que cada proceso deje de tomar jobs nuevos, pero termine los que ya están corriendo. La forma educada de drenar antes de bajar — buena para deploys que no usan orquestrador.

## Deteniendo procesos vía API

```ruby
ps = Sidekiq::ProcessSet.new
ps.each(&:stop!)
```

Equivalente al `sidekiqctl stop` de la [parte 1](/posts/sidekiq-1-infra-iniciando-parando-matando/), solo que por consola en vez de shell.

## Limpiar todo el Redis de Sidekiq — BORRA TODO

```ruby
Sidekiq.redis { |conn| conn.flushdb }
```

Bomba nuclear: borra **todo** lo que Sidekiq tiene en Redis (queues, retries, dead set, stats). Usalo solo cuando sabés que podés reprocesar con tranquilidad o estás en una máquina de desarrollo. En producción, esto es un incidente — solo hacelo con un plan de recuperación claro.

## Próximo en la serie

[Hacks de UI para el panel de Sidekiq](/posts/sidekiq-4-hacks-de-ui-no-painel/) — cuando la pantalla de retries tiene 100 mil ítems y la UI nativa no alcanza.
