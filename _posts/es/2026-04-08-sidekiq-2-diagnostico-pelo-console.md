---
title: "Diagnóstico de Sidekiq por consola: stats, conteos y procesos"
date: 2026-04-08 14:00:00 -0300
updated: 2026-04-08 14:00:00 -0300
tags: [ruby, rails, sidekiq, ops, redis]
excerpt: "Cómo tomar una radiografía de tu instalación Sidekiq sin tocar nada: Sidekiq::Stats, agrupamiento de jobs por clase y listado de procesos y workers."
lang: es
ref: sidekiq-2-diagnostico-pelo-console
---

> **Serie Sidekiq** — parte 2 de 4
> 1. [Infra: iniciando, deteniendo, matando](/posts/sidekiq-1-infra-iniciando-parando-matando/)
> 2. **Estás aquí — Diagnóstico por consola**
> 3. [Manipulación masiva de jobs y procesos](/posts/sidekiq-3-manipulacao-em-massa/)
> 4. [Hacks de UI en el panel](/posts/sidekiq-4-hacks-de-ui-no-painel/)

Cuando algo está raro, antes de salir a borrar o mover jobs, siempre hago una ronda de "lectura" — entender el estado antes de tocar algo. Este post reúne los comandos de solo lectura que uso para ese paso.

## Estado general

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

En 5 segundos sabés: cuántos jobs ya se procesaron, cuántos fallaron, cuál es el tamaño del retry/dead set, cuál es la latencia de la queue por defecto. Es un dashboard mental sin necesitar el panel web.

## Total de jobs en una queue por clase

```ruby
queue_name = 'default'
Sidekiq::Queue.new(queue_name).map { |j| j.klass }.group_by { |e| e }.map { |k, v| [k, v.length] }.to_h
```

Útil para responder "¿qué worker está dominando esta queue?". Cuando la queue se llena, normalmente es una clase específica que está produciendo jobs más rápido de lo que el cluster puede consumir.

## Total de jobs en retry por clase

```ruby
Sidekiq::RetrySet.new.map { |j| j.klass }.group_by { |e| e }.map { |k, v| [k, v.length] }.to_h
```

Muestra qué worker está fallando más. Útil después de un deploy: si una clase nueva aparece dominando el retry set, es una fuerte señal de regresión.

## Total de jobs muertos por clase

```ruby
Sidekiq::DeadSet.new.map { |j| j.klass }.group_by { |e| e }.map { |k, v| [k, v.length] }.to_h
```

El dead set es el cementerio: jobs que agotaron todos los retries. Si ves números creciendo ahí que no eran esperados, vale investigar — literalmente estás perdiendo trabajo.

## Listando procesos en ejecución

```ruby
ps = Sidekiq::ProcessSet.new
ps.size
ps.each do |process|
  p "pid: #{process['pid']} hostname: #{process['hostname']} busy: #{process['busy']} quiet: #{process['quiet']} queues: #{process['queues'].length} [#{process['queues'].sort.join(', ')}]"
end
```

Muestra cada proceso Sidekiq vivo en el cluster con:
- `busy` — cuántos threads están trabajando ahora
- `quiet` — si fue pausado (no toma jobs nuevos)
- `queues` — qué queues consume este proceso

En un cluster con varios nodos, ayuda a entender si uno de los procesos murió sin avisar.

## Inspeccionando workers (threads activos ahora mismo)

```ruby
workers = Sidekiq::Workers.new
workers.size
workers.each do |process_id, thread_id, work|
  p "process_id: #{process_id} thread_id: #{thread_id} queue: #{work['queue']} retry: #{work['payload']['retry']} class: #{work['payload']['class']}"
end
```

`Workers` es diferente de `ProcessSet`: acá ves **threads en ejecución en este mismo momento** — qué clase está corriendo cada uno, en qué queue, si es un intento de retry. Ayuda a entender qué está trabando cuando el contador de processed dejó de subir.

## Próximo en la serie

[Manipulación masiva de jobs y procesos](/posts/sidekiq-3-manipulacao-em-massa/) — seleccionar jobs por clase, mover entre queues, retry/delete masivo, pausar procesos para deploy.
