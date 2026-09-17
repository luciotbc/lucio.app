---
title: "Resque: manipulación masiva de jobs y workers"
date: 2026-04-25 14:00:00 -0300
updated: 2026-04-25 14:00:00 -0300
tags: [ruby, rails, resque, ops, redis]
excerpt: "Snippets para seleccionar failures por clase, requeue/remove en masa, mover jobs entre colas, pausar workers via USR2/CONT, podar workers fantasma y — cuando hace falta — limpiar todo de Redis."
lang: es
ref: resque-3-manipulacao-em-massa
---

> **Serie Resque** — parte 3 de 3
> 1. [Infra: iniciando, deteniendo, matando](/posts/resque-1-infra-iniciando-parando-matando/)
> 2. [Diagnóstico por la consola](/posts/resque-2-diagnostico-pelo-console/)
> 3. **Estás aquí — Manipulación masiva**

> **Correlación con la serie Sidekiq** — este post es el espejo de la [parte 3 de la serie Sidekiq](/posts/sidekiq-3-manipulacao-em-massa/). La gran diferencia es que Resque no tiene `RetrySet`/`DeadSet` — todo lo que falla va al `Resque::Failure`, indexado por posición (no por ID). Esto cambia la forma de hacer "borrado en masa": como remover un item desplaza los índices, hay que iterar de atrás hacia adelante. Este cuidado va a aparecer en casi todos los snippets de abajo.

## Seleccionando failures por clase

```ruby
class_name = 'MyApp::ImportWorker'
total = Resque::Failure.count
failures = Resque::Failure.all(0, total).each_with_index.select { |f, _| f['payload']['class'] == class_name }
failures.size
```

A diferencia de Sidekiq (`RetrySet#select { |j| j.klass == ... }`), la colección de Resque aquí es una lista lineal indexada. La cargo con `each_with_index` para preservar la posición original — la voy a necesitar para retry/delete.

## Seleccionando failures por mensaje de error

```ruby
needle = 'Net::OpenTimeout'
failures = Resque::Failure.all(0, Resque::Failure.count).each_with_index.select { |f, _|
  f['error'].to_s.include?(needle)
}
failures.size
```

Combina bien con el conteo por error del post anterior: descubrís que el 90% de las failures son de un único `OpenTimeout`, aislás solo esas y hacés retry después de que el servicio externo vuelva.

## Reencolando una failure individual

```ruby
Resque::Failure.requeue(0)  # índice 0 es la falla más antigua
```

`requeue` vuelve a poner el job en la cola original sin removerlo del failure set — entonces todavía podés ver el historial después.

## Removiendo una failure individual

```ruby
Resque::Failure.remove(0)
```

Remueve solo esa entrada del failure set. **Importante:** todas las failures después de ella desplazan su índice en 1.

## Retry en masa de una clase — iterando de atrás hacia adelante

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

Este patrón es específico de Resque: como `remove` desplaza índices, **hay que iterar del mayor al menor** para que cada operación no invalide las anteriores. Si hacés `indexes.each` (orden ascendente), vas a borrar/reprogramar el job incorrecto a partir de la segunda iteración.

> Existe `Resque::Failure.requeue_all` y `Resque::Failure.clear` para actuar en todas, sin filtro. Usá cuando el filtro es "todo".

## Borrar todas las failures

```ruby
Resque::Failure.clear
```

Equivalente a `DeadSet#clear`. Sin vuelta atrás — solo hacelo cuando ya confirmaste via diagnóstico (parte 2) que no hay nada útil ahí.

## Moviendo jobs de una clase a una cola dedicada

```ruby
queue_name     = 'default'
new_queue_name = 'isolated_import'
class_name     = 'MyApp::ImportWorker'

snapshot = Resque.peek(queue_name, 0, Resque.size(queue_name))
moved = 0

snapshot.each_with_index do |payload, _|
  next unless payload['class'] == class_name
  # remueve la primera ocurrencia exacta de este payload de la cola
  removed = Resque.redis.lrem("queue:#{queue_name}", 1, Resque.encode(payload))
  if removed > 0
    Resque.push(new_queue_name, payload)
    moved += 1
  end
end

moved
```

Equivalente directo del snippet "Moviendo 1000 jobs de una clase a otra cola" de la [serie Sidekiq](/posts/sidekiq-3-manipulacao-em-massa/). La mecánica es diferente porque Resque guarda los jobs como una `LIST` en Redis (`queue:<nombre>`), entonces usar `LREM` directamente es la forma confiable de remover por payload exacto sin tocar a los vecinos.

Por qué aislar: si una clase está desbordando una cola compartida, en vez de pausar todo, creo una cola dedicada (`isolated_import`), muevo los jobs problemáticos a ella y levanto un worker separado consumiendo solo esa cola. El resto de la operación no lo nota.

```bash
QUEUE=isolated_import COUNT=1 bundle exec rake resque:workers
```

## Borrar una cola entera

```ruby
Resque.remove_queue('isolated_import')
```

Borra tanto el contenido (`queue:<nombre>`) como el registro de la cola en el índice de colas. Útil para una cola creada temporalmente como la de arriba — sin esto, sigue apareciendo en `Resque.queues` aunque esté vacía.

## Pausar todos los workers (USR2)

```ruby
Resque.workers.each do |w|
  hostname, pid, _ = w.id.split(':')
  next unless hostname == Socket.gethostname  # solo workers locales
  Process.kill('USR2', pid.to_i) rescue nil
end
```

Como Resque no tiene API "via Redis" para pausar un worker (a diferencia de `Sidekiq::Process#quiet!`), la única forma es mandar una señal al proceso. Por eso el filtro de hostname: solo podés señalizar workers que corren en la misma máquina donde estás ejecutando la consola.

Para reanudar:

```ruby
Resque.workers.each do |w|
  hostname, pid, _ = w.id.split(':')
  next unless hostname == Socket.gethostname
  Process.kill('CONT', pid.to_i) rescue nil
end
```

> En un cluster multi-host, la alternativa es correr estos snippets via Capistrano/Ansible en cada nodo, u orquestar con systemd. La pausa "por Redis" no existe.

## Podar workers fantasma

```ruby
Resque.workers.each(&:prune_dead_workers)
```

Los workers que murieron sin llamar a `unregister_worker` (por ejemplo, `kill -9` o crash) quedan listados en Redis pero sin proceso correspondiente. `prune_dead_workers` verifica cada worker en el host actual y desregistra los que no tienen proceso vivo. Correlo en cada host periódicamente — en producción, lo dejo en un cron o en el health check de la unit de systemd.

## Limpiar todo el Redis de Resque — BORRA TODO

```ruby
Resque.redis.flushdb
```

Bomba nuclear, equivalente exacto de `Sidekiq.redis { |conn| conn.flushdb }` de la serie anterior: borra **todo** lo que Resque tiene en Redis (colas, workers registrados, failures, stats). Usá solo cuando sabés que podés reprocesar tranquilamente o estás en desarrollo. En producción, esto es un incidente — solo hacelo con un plan de recuperación claro.

> Si Redis tiene otros usos en el mismo db (cache, sessions, otros workers), `flushdb` borra **todo eso también**. Verificá `Resque.redis.client.db` antes — separar Sidekiq/Resque/cache en DBs distintos (`/0`, `/1`, `/2`) es el estándar exactamente para este escenario.

## Final de la serie

Esta fue la parte 3 y la última. Volviendo al principio: [Infra: iniciando, deteniendo, matando](/posts/resque-1-infra-iniciando-parando-matando/).

Y si administrás los dos al mismo tiempo (caso clásico: servicio legacy en Resque + servicios nuevos en Sidekiq), vale tener las dos series lado a lado. La [primera parte de la serie Sidekiq](/posts/sidekiq-1-infra-iniciando-parando-matando/) es el punto de entrada equivalente.
