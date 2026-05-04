---
title: "Diagnóstico de Sidekiq pelo console: stats, contagens e processos"
date: 2026-04-08 14:00:00 -0300
updated: 2026-04-08 14:00:00 -0300
tags: [ruby, rails, sidekiq, ops, redis]
excerpt: "Como tirar uma radiografia da sua instalação Sidekiq sem mexer em nada: Sidekiq::Stats, agrupamento de jobs por classe e listagem de processos e workers."
---

> **Série Sidekiq** — parte 2 de 4
> 1. [Infra: iniciando, parando, matando]({% post_url 2026-04-07-sidekiq-1-infra-iniciando-parando-matando %})
> 2. **Você está aqui — Diagnóstico pelo console**
> 3. [Manipulação em massa de jobs e processos]({% post_url 2026-04-09-sidekiq-3-manipulacao-em-massa %})
> 4. [Hacks de UI no painel]({% post_url 2026-04-10-sidekiq-4-hacks-de-ui-no-painel %})

Quando algo está estranho, antes de sair deletando ou movendo job, eu sempre faço uma rodada de "leitura" — entender o estado antes de mexer. Esse post junta os comandos read-only que uso pra esse passo.

## Status geral

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

Em 5 segundos você sabe: quantos jobs já processou, quantos falharam, qual é o tamanho do retry/dead set, qual a latência da fila padrão. É um dashboard mental sem precisar do painel web.

## Total de jobs em uma fila por classe

```ruby
queue_name = 'default'
Sidekiq::Queue.new(queue_name).map { |j| j.klass }.group_by { |e| e }.map { |k, v| [k, v.length] }.to_h
```

Útil pra responder "qual worker está dominando essa fila?". Quando a fila enche, normalmente é uma classe específica que está produzindo job mais rápido do que o cluster consome.

## Total de jobs em retry por classe

```ruby
Sidekiq::RetrySet.new.map { |j| j.klass }.group_by { |e| e }.map { |k, v| [k, v.length] }.to_h
```

Mostra qual worker está falhando mais. Útil depois de um deploy: se uma classe nova aparece dominando o retry, é forte indício de regressão.

## Total de jobs mortos por classe

```ruby
Sidekiq::DeadSet.new.map { |j| j.klass }.group_by { |e| e }.map { |k, v| [k, v.length] }.to_h
```

Dead set é o cemitério: jobs que esgotaram todos os retries. Se você vê números crescendo aqui que não eram esperados, vale investigar — está literalmente perdendo trabalho.

## Listando processos rodando

```ruby
ps = Sidekiq::ProcessSet.new
ps.size
ps.each do |process|
  p "pid: #{process['pid']} hostname: #{process['hostname']} busy: #{process['busy']} quiet: #{process['quiet']} queues: #{process['queues'].length} [#{process['queues'].sort.join(', ')}]"
end
```

Mostra cada processo Sidekiq vivo no cluster com:
- `busy` — quantas threads estão trabalhando agora
- `quiet` — se foi pausado (não pega jobs novos)
- `queues` — quais filas esse processo consome

Em cluster com vários nós, ajuda a entender se um dos processos morreu sem avisar.

## Inspecionando workers (threads ativas agora)

```ruby
workers = Sidekiq::Workers.new
workers.size
workers.each do |process_id, thread_id, work|
  p "process_id: #{process_id} thread_id: #{thread_id} queue: #{work['queue']} retry: #{work['payload']['retry']} class: #{work['payload']['class']}"
end
```

`Workers` é diferente de `ProcessSet`: aqui você vê **threads em execução neste exato momento** — qual classe cada uma está rodando, em qual fila, se é uma tentativa de retry. Ajuda a entender o que está travando quando o contador de processed parou de subir.

## Próximo da série

[Manipulação em massa de jobs e processos]({% post_url 2026-04-09-sidekiq-3-manipulacao-em-massa %}) — selecionar jobs por classe, mover entre filas, retry/delete em massa, pausar processos pra deploy.
