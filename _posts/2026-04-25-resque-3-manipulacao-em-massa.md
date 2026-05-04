---
title: "Resque: manipulação em massa de jobs e workers"
date: 2026-04-25 14:00:00 -0300
updated: 2026-04-25 14:00:00 -0300
tags: [ruby, rails, resque, ops, redis]
excerpt: "Snippets pra selecionar failures por classe, requeue/remove em massa, mover jobs entre filas, pausar workers via USR2/CONT, prunar workers fantasma e — quando precisa — limpar tudo do Redis."
---

> **Série Resque** — parte 3 de 3
> 1. [Infra: iniciando, parando, matando]({% post_url 2026-04-23-resque-1-infra-iniciando-parando-matando %})
> 2. [Diagnóstico pelo console]({% post_url 2026-04-24-resque-2-diagnostico-pelo-console %})
> 3. **Você está aqui — Manipulação em massa**

> **Correlação com a série Sidekiq** — esse post é o espelho da [parte 3 da série Sidekiq]({% post_url 2026-04-09-sidekiq-3-manipulacao-em-massa %}). A grande diferença é que o Resque não tem `RetrySet`/`DeadSet` — tudo o que falha vai pro `Resque::Failure`, indexado por posição (não por ID). Isso muda o jeito de fazer "delete em massa": como remover um item desloca os índices, é preciso iterar de trás pra frente. Esse cuidado vai aparecer em quase todo snippet abaixo.

## Selecionando failures por classe

```ruby
class_name = 'MyApp::ImportWorker'
total = Resque::Failure.count
failures = Resque::Failure.all(0, total).each_with_index.select { |f, _| f['payload']['class'] == class_name }
failures.size
```

Diferente do Sidekiq (`RetrySet#select { |j| j.klass == ... }`), aqui a coleção do Resque é uma lista linear indexada. Carrego com `each_with_index` pra preservar a posição original — vou precisar dela pra retry/delete.

## Selecionando failures por mensagem de erro

```ruby
needle = 'Net::OpenTimeout'
failures = Resque::Failure.all(0, Resque::Failure.count).each_with_index.select { |f, _|
  f['error'].to_s.include?(needle)
}
failures.size
```

Combina bem com a contagem por erro do post anterior: você descobre que 90% das failures são de um único `OpenTimeout`, isola só elas e dá retry depois que o serviço externo voltou.

## Reenfileirando uma failure individual

```ruby
Resque::Failure.requeue(0)  # índice 0 é a falha mais antiga
```

`requeue` recoloca o job na fila original sem remover do failure set — então você ainda vê o histórico depois.

## Removendo uma failure individual

```ruby
Resque::Failure.remove(0)
```

Remove só essa entrada do failure set. **Importante:** todas as failures depois dela deslocam o índice em 1.

## Retry em massa de uma classe — iterando de trás pra frente

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

Esse padrão é específico do Resque: como `remove` desloca índices, **tem que iterar do maior pro menor** pra cada operação não invalidar as anteriores. Se você fizer `indexes.each` (ordem crescente), vai apagar/reagendar o job errado a partir da segunda iteração.

> Existe `Resque::Failure.requeue_all` e `Resque::Failure.clear` pra agir em todas, sem filtro. Use quando o filtro é "tudo".

## Apagar todas as failures

```ruby
Resque::Failure.clear
```

Equivalente ao `DeadSet#clear`. Sem volta — só faça quando já confirmou via diagnóstico (parte 2) que não tem nada útil ali.

## Movendo jobs de uma classe pra fila dedicada

```ruby
queue_name     = 'default'
new_queue_name = 'isolated_import'
class_name     = 'MyApp::ImportWorker'

snapshot = Resque.peek(queue_name, 0, Resque.size(queue_name))
moved = 0

snapshot.each_with_index do |payload, _|
  next unless payload['class'] == class_name
  # remove a primeira ocorrência exata desse payload da fila
  removed = Resque.redis.lrem("queue:#{queue_name}", 1, Resque.encode(payload))
  if removed > 0
    Resque.push(new_queue_name, payload)
    moved += 1
  end
end

moved
```

Equivalente direto do snippet "Movendo 1000 jobs de uma classe pra outra fila" da [série Sidekiq]({% post_url 2026-04-09-sidekiq-3-manipulacao-em-massa %}). A mecânica é diferente porque o Resque guarda os jobs como uma `LIST` no Redis (`queue:<nome>`), então usar `LREM` direto é a forma confiável de remover por payload exato sem mexer nos vizinhos.

Por que isolar: se uma classe está derrubando uma fila compartilhada, em vez de pausar tudo, eu crio uma fila dedicada (`isolated_import`), movo os jobs problemáticos pra ela e subo um worker separado consumindo só essa fila. O resto da operação não sente.

```bash
QUEUE=isolated_import COUNT=1 bundle exec rake resque:workers
```

## Apagar uma fila inteira

```ruby
Resque.remove_queue('isolated_import')
```

Apaga tanto o conteúdo (`queue:<nome>`) quanto o registro da fila no índice de filas. Útil pra fila criada temporariamente como acima — sem isso, ela continua aparecendo no `Resque.queues` mesmo vazia.

## Pausar todos os workers (USR2)

```ruby
Resque.workers.each do |w|
  hostname, pid, _ = w.id.split(':')
  next unless hostname == Socket.gethostname  # só workers locais
  Process.kill('USR2', pid.to_i) rescue nil
end
```

Como Resque não tem API "via Redis" pra pausar um worker (diferente do `Sidekiq::Process#quiet!`), a única forma é mandar sinal pro processo. Por isso o filtro de hostname: só dá pra sinalizar workers que rodam na mesma máquina onde você está executando o console.

Pra resumir:

```ruby
Resque.workers.each do |w|
  hostname, pid, _ = w.id.split(':')
  next unless hostname == Socket.gethostname
  Process.kill('CONT', pid.to_i) rescue nil
end
```

> Em cluster multi-host, a alternativa é rodar esses snippets via Capistrano/Ansible em cada nó, ou orquestrar com systemd. Pausa "pelo Redis" não existe.

## Prunar workers fantasma

```ruby
Resque.workers.each(&:prune_dead_workers)
```

Workers que morreram sem chamar `unregister_worker` (ex: `kill -9` ou crash) ficam listados no Redis mas sem processo correspondente. `prune_dead_workers` checa cada worker no host atual e desregistra os que não têm processo vivo. Roda em cada host periodicamente — em produção, eu deixo num cron ou no health check da unit do systemd.

## Limpar todo o Redis do Resque — APAGA TUDO

```ruby
Resque.redis.flushdb
```

Bomba nuclear, equivalente exato do `Sidekiq.redis { |conn| conn.flushdb }` da série anterior: apaga **tudo** que o Resque tem no Redis (filas, workers registrados, failures, stats). Use só quando sabe que pode reprocessar com tranquilidade ou está em dev. Em produção, isso é incidente — só faça com plano de recuperação claro.

> Se o Redis tem outros usos no mesmo db (cache, sessions, outros workers), `flushdb` apaga **tudo isso também**. Verifique `Resque.redis.client.db` antes — separar Sidekiq/Resque/cache em DBs diferentes (`/0`, `/1`, `/2`) é o padrão exatamente pra esse cenário.

## Final da série

Essa foi a parte 3 e última. Voltando ao começo: [Infra: iniciando, parando, matando]({% post_url 2026-04-23-resque-1-infra-iniciando-parando-matando %}).

E se você administra os dois ao mesmo tempo (caso clássico: serviço legado em Resque + serviços novos em Sidekiq), vale ter as duas séries lado a lado. A [primeira parte da série Sidekiq]({% post_url 2026-04-07-sidekiq-1-infra-iniciando-parando-matando %}) é o ponto de entrada equivalente.
