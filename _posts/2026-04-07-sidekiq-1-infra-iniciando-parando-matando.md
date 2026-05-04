---
title: "Sidekiq na operação: iniciando, parando e matando processos"
date: 2026-04-07 14:00:00 -0300
updated: 2026-04-07 14:00:00 -0300
tags: [ruby, rails, sidekiq, ops]
excerpt: "Comandos de shell pra subir, derrubar com graça e (último recurso) matar à força um processo Sidekiq. Por que sidekiqctl stop primeiro e quando partir pro kill -9."
---

> **Série Sidekiq** — parte 1 de 4
> 1. **Você está aqui — Infra: iniciando, parando, matando**
> 2. [Diagnóstico pelo console]({% post_url 2026-04-08-sidekiq-2-diagnostico-pelo-console %})
> 3. [Manipulação em massa de jobs e processos]({% post_url 2026-04-09-sidekiq-3-manipulacao-em-massa %})
> 4. [Hacks de UI no painel]({% post_url 2026-04-10-sidekiq-4-hacks-de-ui-no-painel %})

Antes de mergulhar nas filas e jobs, é bom ter o básico do ciclo de vida do processo Sidekiq sólido — porque a maior parte dos incidentes começa com "como derrubo isso sem perder jobs".

Documentação da API: [github.com/mperham/sidekiq/wiki/API](https://github.com/mperham/sidekiq/wiki/API).

## Iniciando o Sidekiq

```bash
bundle exec sidekiq -d -L log/sidekiq.log -C config/sidekiq.yml
```

- `-d` daemoniza (libera o terminal).
- `-L log/sidekiq.log` aponta o arquivo de log.
- `-C config/sidekiq.yml` aponta o arquivo de configuração (filas, concorrência, retries).

Em produção esse comando geralmente vive num systemd unit ou no Procfile do app, mas pra testes locais e VPS sem orquestrador é perfeito.

## Parando pelo controlador do Rails (recomendado)

```bash
ps -ef | grep sidekiq | grep busy | grep -v grep | awk '{print $2}' > tmp/sidekiq.pid
cat tmp/sidekiq.pid
bundle exec sidekiqctl stop tmp/sidekiq.pid
```

`sidekiqctl stop` espera os jobs em execução terminarem (até o timeout configurado) antes de matar o processo. É a forma educada — preserva idempotência se algum worker estiver no meio de uma operação não-atômica.

O pipe `ps -ef | grep busy` filtra pelo processo que está realmente trabalhando (não a master), e o `awk '{print $2}'` extrai só o PID.

## Parando pelo SO (último recurso)

```bash
ps -ef | grep sidekiq | grep busy | grep -v grep | awk '{print $2}'
kill -9 $(ps -ef | grep sidekiq | grep busy | grep -v grep | awk '{print $2}')
```

`kill -9` é o botão vermelho: o processo morre na hora, sem chance de terminar o que estava fazendo. Job em execução vira retry, então só faça quando:

- O `sidekiqctl stop` travou e não responde
- O processo virou zumbi e está consumindo memória sem fazer nada
- Você está numa situação de incêndio onde derrubar é mais importante que preservar idempotência

Se você se pega usando `kill -9` com frequência, vale investigar por que o `stop` está demorando — geralmente é job que não respeita o timeout ou conexão de banco travada.

## Próximo da série

[Diagnóstico de Sidekiq pelo console]({% post_url 2026-04-08-sidekiq-2-diagnostico-pelo-console %}) — `Sidekiq::Stats`, agrupamento por classe, listagem de processos e workers.
