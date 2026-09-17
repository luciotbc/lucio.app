---
title: "Sidekiq en producción: iniciando, deteniendo y matando procesos"
date: 2026-04-07 14:00:00 -0300
updated: 2026-04-07 14:00:00 -0300
tags: [ruby, rails, sidekiq, ops]
excerpt: "Comandos de shell para levantar, detener con gracia y (como último recurso) matar a la fuerza un proceso Sidekiq. Por qué usar sidekiqctl stop primero y cuándo recurrir a kill -9."
lang: es
ref: sidekiq-1-infra-iniciando-parando-matando
---

> **Serie Sidekiq** — parte 1 de 4
> 1. **Estás aquí — Infra: iniciando, deteniendo, matando**
> 2. [Diagnóstico por consola](/posts/sidekiq-2-diagnostico-pelo-console/)
> 3. [Manipulación masiva de jobs y procesos](/posts/sidekiq-3-manipulacao-em-massa/)
> 4. [Hacks de UI en el panel](/posts/sidekiq-4-hacks-de-ui-no-painel/)

Antes de sumergirse en las queues y jobs, conviene tener bien claro el ciclo de vida básico del proceso Sidekiq — porque la mayoría de los incidentes empiezan con "¿cómo bajo esto sin perder jobs?".

Documentación de la API: [github.com/mperham/sidekiq/wiki/API](https://github.com/mperham/sidekiq/wiki/API).

## Iniciando Sidekiq

```bash
bundle exec sidekiq -d -L log/sidekiq.log -C config/sidekiq.yml
```

- `-d` daemoniza (libera la terminal).
- `-L log/sidekiq.log` indica el archivo de log.
- `-C config/sidekiq.yml` indica el archivo de configuración (queues, concurrencia, retries).

En producción, este comando generalmente vive en un systemd unit o en el Procfile de la aplicación, pero para pruebas locales y VPS sin orquestador es perfecto.

## Detener mediante el controlador de Rails (recomendado)

```bash
ps -ef | grep sidekiq | grep busy | grep -v grep | awk '{print $2}' > tmp/sidekiq.pid
cat tmp/sidekiq.pid
bundle exec sidekiqctl stop tmp/sidekiq.pid
```

`sidekiqctl stop` espera a que los jobs en ejecución terminen (hasta el timeout configurado) antes de matar el proceso. Es la forma educada — preserva la idempotencia si algún worker está en medio de una operación no atómica.

El pipe `ps -ef | grep busy` filtra por el proceso que realmente está trabajando (no el master), y `awk '{print $2}'` extrae solo el PID.

## Detener mediante el SO (último recurso)

```bash
ps -ef | grep sidekiq | grep busy | grep -v grep | awk '{print $2}'
kill -9 $(ps -ef | grep sidekiq | grep busy | grep -v grep | awk '{print $2}')
```

`kill -9` es el botón rojo: el proceso muere al instante, sin oportunidad de terminar lo que estaba haciendo. Un job en ejecución se convierte en retry, así que úsalo solo cuando:

- El `sidekiqctl stop` se trabó y no responde
- El proceso se convirtió en zombie y está consumiendo memoria sin hacer nada
- Estás en una situación de emergencia donde derribar el proceso es más importante que preservar la idempotencia

Si te encontrás usando `kill -9` con frecuencia, vale la pena investigar por qué el `stop` está tardando — generalmente es un job que no respeta el timeout o una conexión de base de datos trabada.

## Próximo en la serie

[Diagnóstico de Sidekiq por consola](/posts/sidekiq-2-diagnostico-pelo-console/) — `Sidekiq::Stats`, agrupamiento por clase, listado de procesos y workers.
