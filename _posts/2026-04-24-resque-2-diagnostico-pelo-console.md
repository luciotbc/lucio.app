---
title: "Diagnóstico de Resque pelo console: info, contagens e workers"
date: 2026-04-24 14:00:00 -0300
updated: 2026-04-24 14:00:00 -0300
tags: [ruby, rails, resque, ops, redis]
excerpt: "Tirando uma radiografia da instalação Resque sem mexer em nada: Resque.info, contagem de jobs por classe em filas, failure set e listagem do que cada worker está executando agora."
---

> **Série Resque** — parte 2 de 3
> 1. [Infra: iniciando, parando, matando]({% post_url 2026-04-23-resque-1-infra-iniciando-parando-matando %})
> 2. **Você está aqui — Diagnóstico pelo console**
> 3. [Manipulação em massa de jobs e workers]({% post_url 2026-04-25-resque-3-manipulacao-em-massa %})

> **Correlação com a série Sidekiq** — esse post é o espelho da [parte 2 da série Sidekiq]({% post_url 2026-04-08-sidekiq-2-diagnostico-pelo-console %}). Mesma filosofia: antes de sair deletando ou movendo job, fazer uma rodada read-only pra entender o estado. As APIs são diferentes (Resque expõe muita coisa via `Resque.info` e classes auxiliares em vez de `Sidekiq::Stats`/`Sidekiq::Queue`), mas as perguntas que você quer responder são as mesmas: o que está pendente, o que está falhando, e quem está trabalhando agora.

## Status geral

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

Em uma chamada você tem o panorama: pendente (soma de todas as filas), total processado historicamente, número de filas, workers vivos, workers ativos *agora* e total de failures acumuladas. É o equivalente do `Sidekiq::Stats.new` da série anterior.

## Listando filas e tamanhos

```ruby
Resque.queues
# => ["default", "critical", "mailers", "low"]

Resque.queues.map { |q| [q, Resque.size(q)] }.to_h
# => {"default"=>120, "critical"=>0, "mailers"=>3, "low"=>15000}
```

`Resque.size('queue_name')` é O(1) — usa `LLEN` no Redis. Pode chamar à vontade.

## Total de jobs em uma fila por classe

```ruby
queue_name = 'default'
Resque.peek(queue_name, 0, Resque.size(queue_name))
  .group_by { |job| job['class'] }
  .map { |k, v| [k, v.length] }
  .to_h
```

`Resque.peek(queue, start, count)` devolve os payloads sem remover (LRANGE no Redis). Como o payload é um Hash já parseado, basta agrupar por `'class'`.

Útil pra responder a mesma pergunta da série Sidekiq: "qual worker está dominando essa fila?". Em incidente pós-deploy, normalmente é uma classe específica produzindo job mais rápido do que o cluster consome.

> Cuidado: se a fila tem milhões de itens, evite carregar tudo. Pegue uma amostra (`Resque.peek(queue, 0, 5000)`) — geralmente já dá pra inferir a distribuição.

## Failures por classe

Resque não separa "retry" e "dead" como o Sidekiq — toda falha vai pro `failure backend` (geralmente o Redis). A leitura é parecida:

```ruby
total = Resque::Failure.count
Resque::Failure.all(0, total)
  .group_by { |f| f['payload']['class'] }
  .map { |k, v| [k, v.length] }
  .to_h
```

Mostra qual classe está dominando o failure set. Depois de um deploy ruim, se uma classe nova aparece dominando, é forte indício de regressão — mesma heurística da [parte 2 do Sidekiq]({% post_url 2026-04-08-sidekiq-2-diagnostico-pelo-console %}).

## Failures por mensagem de erro

Uma vantagem do Resque: o failure guarda a exceção e a mensagem direto no payload, então dá pra agrupar por erro também:

```ruby
Resque::Failure.all(0, Resque::Failure.count)
  .group_by { |f| "#{f['exception']}: #{f['error']}" }
  .map { |k, v| [k, v.length] }
  .sort_by { |_, v| -v }
  .first(10)
```

Top 10 erros mais frequentes. Em incidente, isso responde "todas as 5000 falhas são do mesmo `Net::OpenTimeout` ou tem coisa nova no meio?" em segundos.

## Listando workers (todos os processos vivos)

```ruby
Resque.workers.each do |w|
  puts "#{w.to_s} | host=#{w.hostname} pid=#{w.pid} queues=#{w.queues.join(',')}"
end
```

Cada worker se registra no Redis quando sobe e desregistra quando sai limpo. Se você vê um worker listado mas o processo não existe mais (caiu sem QUIT), é um *worker fantasma* — abordo isso na próxima parte.

## O que cada worker está executando agora

```ruby
Resque.working.each do |w|
  job = w.job
  puts "#{w.to_s} | class=#{job['payload'] && job['payload']['class']} queue=#{job['queue']} run_at=#{job['run_at']}"
end
```

`Resque.working` filtra só os que têm job em mãos. Equivalente ao `Sidekiq::Workers.new` da série anterior — mostra threads/processos em execução **neste exato momento**, qual classe e em qual fila.

Quando o `Resque.info[:processed]` parou de subir mas os workers ainda estão "working", é aqui que você descobre quem travou.

## Tempo que cada worker está em um job

```ruby
require 'time'

Resque.working.each do |w|
  job = w.job
  next if job.empty?
  run_at = Time.parse(job['run_at'])
  puts "#{w.to_s} -> #{job['payload']['class']} rodando há #{(Time.now - run_at).to_i}s"
end
```

Acha worker travado: se o `rodando há` está em milhares de segundos pra um job que deveria levar 200ms, alguém ficou pendurado em I/O.

## Workers fantasma

```ruby
Resque.workers.reject { |w|
  hostname, pid, _ = w.id.split(':')
  hostname == `hostname`.strip && system("ps -p #{pid} > /dev/null 2>&1")
}
```

Lista workers registrados no Redis cujo processo não existe mais na máquina onde rodam. Em cluster com vários hosts, esse check só vale pros workers locais — pra cluster real, melhor confiar no `prune_dead_workers` (que vou usar na parte 3).

## Próximo da série

[Manipulação em massa de jobs e workers]({% post_url 2026-04-25-resque-3-manipulacao-em-massa %}) — selecionar failures por classe, requeue/remove em massa, mover entre filas, pausar workers via sinal e a bomba nuclear `Resque.redis.flushdb`.
