---
title: "Sidekiq: manipulação em massa de jobs e processos"
date: 2026-04-09 14:00:00 -0300
updated: 2026-04-09 14:00:00 -0300
tags: [ruby, rails, sidekiq, ops, redis]
excerpt: "Snippets pra selecionar jobs por classe, mover entre filas, dar retry ou delete em massa, pausar processos pra deploy e — quando precisa — limpar tudo do Redis."
---

> **Série Sidekiq** — parte 3 de 4
> 1. [Infra: iniciando, parando, matando]({% post_url 2026-04-07-sidekiq-1-infra-iniciando-parando-matando %})
> 2. [Diagnóstico pelo console]({% post_url 2026-04-08-sidekiq-2-diagnostico-pelo-console %})
> 3. **Você está aqui — Manipulação em massa**
> 4. [Hacks de UI no painel]({% post_url 2026-04-10-sidekiq-4-hacks-de-ui-no-painel %})

Depois de diagnosticar (parte 2), normalmente vem a parte invasiva: tirar um worker problemático da frente, reorganizar a fila, deletar jobs que não fazem mais sentido. Esse post é a caixa de ferramentas pra isso.

> O `; nil` no final dos blocos é só pra evitar o `rails console` imprimir milhares de linhas quando a coleção é grande.

## Selecionando jobs em retry pelo nome da classe

```ruby
job_class_name = 'SidekiqTest::SidekiqTestWorker'
jobs = Sidekiq::RetrySet.new.select { |job| job.klass == job_class_name }; nil
jobs.size
```

## Selecionando jobs mortos pelo nome da classe

```ruby
job_class_name = 'SidekiqTest::SidekiqTestWorker'
jobs = Sidekiq::DeadSet.new.select { |job| job.klass == job_class_name }; nil
jobs.size
```

## Selecionando jobs enfileirados em uma fila pelo nome da classe

```ruby
queue_name = 'default'
job_class_name = 'SidekiqTest::SidekiqTestWorker'
jobs = Sidekiq::Queue.new(queue_name).select { |job| job.klass == job_class_name }; nil
jobs.count
```

## Reenfileirando jobs selecionados pra retry

```ruby
jobs.each(&:retry)
```

Aplica em qualquer coleção de jobs vinda do `RetrySet` ou `DeadSet`. Move tudo de volta pra fila de origem.

## Deletando jobs selecionados

```ruby
jobs.each(&:delete)
```

Mesmo padrão, com a operação inversa. Use depois que você tem certeza que pode descartar — não tem volta.

## Movendo 1000 jobs de uma classe pra outra fila

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

Esse é o snippet que mais salva o dia. Quando um worker está derrubando uma fila compartilhada, em vez de pausar tudo, eu:

1. Crio uma fila dedicada (`funnels_test_worker`)
2. Movo os jobs problemáticos pra ela
3. Subo um processo Sidekiq separado consumindo essa fila com concorrência reduzida (ex: 1 thread)

O resto da operação não sente — e eu posso debugar o worker com calma sem pressionar a fila principal.

## Pausando processos pra deploy (quiet)

```ruby
ps = Sidekiq::ProcessSet.new
ps.each(&:quiet!)
```

`quiet!` faz cada processo parar de pegar jobs novos, mas terminar os que estão rodando. Forma educada de drenar antes de derrubar — bom pra deploys que não usam orquestrador.

## Parando processos via API

```ruby
ps = Sidekiq::ProcessSet.new
ps.each(&:stop!)
```

Equivalente ao `sidekiqctl stop` da [parte 1]({% post_url 2026-04-07-sidekiq-1-infra-iniciando-parando-matando %}), só que via console em vez de shell.

## Limpar todo o Redis do Sidekiq — APAGA TUDO

```ruby
Sidekiq.redis { |conn| conn.flushdb }
```

Bomba nuclear: apaga **tudo** que o Sidekiq tem no Redis (filas, retries, dead set, stats). Use só quando você sabe que pode reprocessar com tranquilidade ou está numa máquina de dev. Em produção, isso é incidente — só faça com plano de recuperação claro.

## Próximo da série

[Hacks de UI pro painel do Sidekiq]({% post_url 2026-04-10-sidekiq-4-hacks-de-ui-no-painel %}) — quando a tela de retries tem 100 mil itens e a UI nativa não dá conta.
