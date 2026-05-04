---
title: "Resque na operação: iniciando, parando e matando workers"
date: 2026-04-23 14:00:00 -0300
updated: 2026-04-23 14:00:00 -0300
tags: [ruby, rails, resque, ops, redis]
excerpt: "Como subir, drenar com graça e (último recurso) matar à força um worker Resque. Os sinais QUIT, TERM, USR1, USR2 e CONT — e quando usar cada um."
---

> **Série Resque** — parte 1 de 3
> 1. **Você está aqui — Infra: iniciando, parando, matando**
> 2. [Diagnóstico pelo console]({% post_url 2026-04-24-resque-2-diagnostico-pelo-console %})
> 3. [Manipulação em massa de jobs e workers]({% post_url 2026-04-25-resque-3-manipulacao-em-massa %})

> **Por que essa série existe** — trabalhei muito tempo em projetos de empresas que usavam Sidekiq e, durante esse processo, fui compilando minhas anotações na [série de Sidekiq]({% post_url 2026-04-07-sidekiq-1-infra-iniciando-parando-matando %}). Depois acabei entrando em um projeto que usa Resque — isso gerou novas anotações, que mantive na mesma estrutura das que eu já havia feito pro Sidekiq. Os problemas operacionais são os mesmos (filas que enchem, jobs que falham em lote, deploy que precisa drenar worker), mas as ferramentas mudam bastante. Esta série é o espelho daquela nos temas que se traduzem bem: infra, diagnóstico e manipulação em massa, com os comandos equivalentes em Resque. Quando algum tópico tem paralelo direto, vou linkar pro post correspondente da série Sidekiq.

A primeira diferença importante de modelo: **Resque é process-per-job, Sidekiq é thread-per-job**. Cada worker Resque ouve uma ou mais filas e, ao pegar um job, faz `fork` de um processo filho que executa o job e morre. Isso muda como você inicia, drena e mata — porque há sempre dois processos relacionados (o worker pai e o filho que está executando).

Documentação da API: [github.com/resque/resque](https://github.com/resque/resque).

## Iniciando o Resque

```bash
QUEUE=* bundle exec rake resque:work
```

- `QUEUE=*` faz o worker consumir todas as filas que existirem no Redis.
- `QUEUE=critical,high,default` ouve em ordem de prioridade — esvazia `critical` antes de tocar em `high`.

Subindo vários workers numa máquina só:

```bash
COUNT=5 QUEUE=* bundle exec rake resque:workers
```

Em background com PID file (útil pra `kill` depois):

```bash
PIDFILE=./tmp/pids/resque.pid BACKGROUND=yes QUEUE=* bundle exec rake resque:work
```

Em produção isso vive em systemd ou foreman, mas pra VPS ou ambiente de teste o `BACKGROUND=yes + PIDFILE` cobre.

> Diferente do Sidekiq, não existe `-C config/sidekiq.yml`. Concorrência em Resque vem de subir N processos (não threads), então a "configuração" é quantos `rake resque:work` você tem rodando.

## Drenando com graça (QUIT — recomendado)

```bash
kill -QUIT $(cat tmp/pids/resque.pid)
```

`QUIT` é o sinal educado: o worker espera o job atual terminar (no processo filho), depois sai limpo. Equivalente moral ao `sidekiqctl stop` da [parte 1 da série Sidekiq]({% post_url 2026-04-07-sidekiq-1-infra-iniciando-parando-matando %}) — preserva idempotência se o worker está no meio de uma operação não-atômica.

Se você não tem o PID file, dá pra extrair pelo `ps`:

```bash
ps -ef | grep '[r]esque' | grep -v 'master' | awk '{print $2}'
kill -QUIT $(ps -ef | grep '[r]esque' | grep -v 'master' | awk '{print $2}')
```

O `[r]esque` é o truque clássico pra evitar que o próprio `grep` apareça no resultado.

## Matando o filho mas mantendo o worker (TERM/USR1)

Como Resque faz fork por job, você pode matar **só o job em execução** sem derrubar o worker pai:

```bash
# TERM no worker: mata o filho na hora E encerra o worker
kill -TERM $(cat tmp/pids/resque.pid)

# USR1 no worker: mata o filho na hora, mas o worker continua e pega o próximo job
kill -USR1 $(cat tmp/pids/resque.pid)
```

`USR1` é a ferramenta certa pra "esse job específico travou e está segurando o worker, mas eu não quero derrubar a infra". O job vira failure, o worker volta pro loop.

## Pausando sem matar (USR2 / CONT)

```bash
# USR2: para de pegar jobs novos (não termina o atual, só não pega o próximo)
kill -USR2 $(cat tmp/pids/resque.pid)

# CONT: volta a pegar jobs
kill -CONT $(cat tmp/pids/resque.pid)
```

Esse par é o equivalente Resque do `Sidekiq::ProcessSet#quiet!` que aparece na [parte 3 da série Sidekiq]({% post_url 2026-04-09-sidekiq-3-manipulacao-em-massa %}). Útil pra deploy: `USR2` em todos os workers, espera os jobs em curso terminarem, faz o deploy, `CONT` pra voltar.

## Último recurso (KILL)

```bash
kill -9 $(cat tmp/pids/resque.pid)
```

`KILL` (`-9`) mata o worker e deixa órfão qualquer filho em execução — esses filhos viram processos zumbi até o init reapeá-los, e o job rodando some sem virar failure (porque o worker não teve tempo de registrar). Use só quando:

- `QUIT` travou e não responde
- O worker está consumindo memória sem fazer nada visível
- Está acontecendo incêndio e derrubar é mais importante que preservar estado

## Resumo dos sinais

| Sinal | Efeito |
|-------|--------|
| `QUIT` | Termina jobs em curso, depois sai (graceful) |
| `TERM` | Mata filho na hora e sai |
| `USR1` | Mata filho na hora, mantém worker rodando |
| `USR2` | Para de pegar jobs novos (pause) |
| `CONT` | Volta a pegar jobs (resume) |
| `KILL` (-9) | Mata o worker, deixa filhos órfãos |

## Próximo da série

[Diagnóstico de Resque pelo console]({% post_url 2026-04-24-resque-2-diagnostico-pelo-console %}) — `Resque.info`, contagens de filas, listagem de workers e do que cada um está fazendo.
