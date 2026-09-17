---
title: "Diagnóstico de Resque por la consola: info, conteos y workers"
date: 2026-04-24 14:00:00 -0300
updated: 2026-04-24 14:00:00 -0300
tags: [ruby, rails, resque, ops, redis]
excerpt: "Tomando una radiografía de la instalación de Resque sin tocar nada: Resque.info, conteo de jobs por clase en colas, el failure set y listado de lo que cada worker está ejecutando ahora mismo."
lang: es
ref: resque-2-diagnostico-pelo-console
---

> **Serie Resque** — parte 2 de 3
> 1. [Infra: iniciando, deteniendo, matando](/posts/resque-1-infra-iniciando-parando-matando/)
> 2. **Estás aquí — Diagnóstico por la consola**
> 3. [Manipulación masiva de jobs y workers](/posts/resque-3-manipulacao-em-massa/)

> **Correlación con la serie Sidekiq** — este post es el espejo de la [parte 2 de la serie Sidekiq](/posts/sidekiq-2-diagnostico-pelo-console/). La misma filosofía: antes de salir a borrar o mover jobs, hacer una ronda de solo lectura para entender el estado. Las APIs son diferentes (Resque expone mucho a través de `Resque.info` y clases auxiliares en vez de `Sidekiq::Stats`/`Sidekiq::Queue`), pero las preguntas que querés responder son las mismas: qué está pendiente, qué está fallando, y quién está trabajando ahora mismo.

## Estado general

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

En una sola llamada tenés el panorama completo: pendiente (suma de todas las colas), total procesado históricamente, número de colas, workers vivos, workers activos *ahora mismo* y total de failures acumuladas. Es el equivalente de `Sidekiq::Stats.new` de la serie anterior.

## Listando colas y tamaños

```ruby
Resque.queues
# => ["default", "critical", "mailers", "low"]

Resque.queues.map { |q| [q, Resque.size(q)] }.to_h
# => {"default"=>120, "critical"=>0, "mailers"=>3, "low"=>15000}
```

`Resque.size('queue_name')` es O(1) — usa `LLEN` en Redis. Podés llamarlo libremente.

## Total de jobs en una cola por clase

```ruby
queue_name = 'default'
Resque.peek(queue_name, 0, Resque.size(queue_name))
  .group_by { |job| job['class'] }
  .map { |k, v| [k, v.length] }
  .to_h
```

`Resque.peek(queue, start, count)` devuelve los payloads sin removerlos (LRANGE en Redis). Como el payload es un Hash ya parseado, basta con agrupar por `'class'`.

Útil para responder la misma pregunta de la serie Sidekiq: "¿qué worker está dominando esta cola?". En un incidente post-deploy, normalmente es una clase específica produciendo jobs más rápido de lo que el cluster puede consumir.

> Cuidado: si la cola tiene millones de items, evitá cargar todo. Tomá una muestra (`Resque.peek(queue, 0, 5000)`) — generalmente ya alcanza para inferir la distribución.

## Failures por clase

Resque no separa "retry" y "dead" como Sidekiq — toda falla va al `failure backend` (generalmente Redis). La lectura es similar:

```ruby
total = Resque::Failure.count
Resque::Failure.all(0, total)
  .group_by { |f| f['payload']['class'] }
  .map { |k, v| [k, v.length] }
  .to_h
```

Muestra qué clase está dominando el failure set. Después de un deploy malo, si una clase nueva aparece dominando, es un fuerte indicio de regresión — la misma heurística de la [parte 2 de Sidekiq](/posts/sidekiq-2-diagnostico-pelo-console/).

## Failures por mensaje de error

Una ventaja de Resque: la failure guarda la excepción y el mensaje directo en el payload, entonces se puede agrupar por error también:

```ruby
Resque::Failure.all(0, Resque::Failure.count)
  .group_by { |f| "#{f['exception']}: #{f['error']}" }
  .map { |k, v| [k, v.length] }
  .sort_by { |_, v| -v }
  .first(10)
```

Top 10 errores más frecuentes. En un incidente, esto responde "¿todas las 5000 fallas son del mismo `Net::OpenTimeout` o hay algo nuevo mezclado?" en segundos.

## Listando workers (todos los procesos vivos)

```ruby
Resque.workers.each do |w|
  puts "#{w.to_s} | host=#{w.hostname} pid=#{w.pid} queues=#{w.queues.join(',')}"
end
```

Cada worker se registra en Redis cuando levanta y se desregistra cuando sale limpiamente. Si ves un worker listado pero el proceso ya no existe (cayó sin QUIT), es un *worker fantasma* — lo abordo en la próxima parte.

## Lo que cada worker está ejecutando ahora mismo

```ruby
Resque.working.each do |w|
  job = w.job
  puts "#{w.to_s} | class=#{job['payload'] && job['payload']['class']} queue=#{job['queue']} run_at=#{job['run_at']}"
end
```

`Resque.working` filtra solo los que tienen un job en mano. Equivalente a `Sidekiq::Workers.new` de la serie anterior — muestra threads/procesos en ejecución **en este exacto momento**, qué clase y en qué cola.

Cuando `Resque.info[:processed]` dejó de subir pero los workers siguen "working", es acá donde descubrís quién se trabó.

## Tiempo que cada worker lleva en un job

```ruby
require 'time'

Resque.working.each do |w|
  job = w.job
  next if job.empty?
  run_at = Time.parse(job['run_at'])
  puts "#{w.to_s} -> #{job['payload']['class']} corriendo hace #{(Time.now - run_at).to_i}s"
end
```

Encuentra workers trabados: si el `corriendo hace` está en miles de segundos para un job que debería tomar 200ms, alguien quedó colgado en I/O.

## Workers fantasma

```ruby
Resque.workers.reject { |w|
  hostname, pid, _ = w.id.split(':')
  hostname == `hostname`.strip && system("ps -p #{pid} > /dev/null 2>&1")
}
```

Lista workers registrados en Redis cuyo proceso ya no existe en la máquina donde corren. En un cluster con varios hosts, este check solo vale para los workers locales — para un cluster real, es mejor confiar en `prune_dead_workers` (que voy a usar en la parte 3).

## Próximo en la serie

[Manipulación masiva de jobs y workers](/posts/resque-3-manipulacao-em-massa/) — seleccionar failures por clase, requeue/remove en masa, mover entre colas, pausar workers via señal y la bomba nuclear `Resque.redis.flushdb`.
