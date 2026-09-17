---
title: "Resque en operaciones: iniciando, deteniendo y matando workers"
date: 2026-04-23 14:00:00 -0300
updated: 2026-04-23 14:00:00 -0300
tags: [ruby, rails, resque, ops, redis]
excerpt: "Cómo levantar, drenar con gracia y (como último recurso) matar a la fuerza un worker de Resque. Las señales QUIT, TERM, USR1, USR2 y CONT — y cuándo usar cada una."
lang: es
ref: resque-1-infra-iniciando-parando-matando
---

> **Serie Resque** — parte 1 de 3
> 1. **Estás aquí — Infra: iniciando, deteniendo, matando**
> 2. [Diagnóstico por la consola](/posts/resque-2-diagnostico-pelo-console/)
> 3. [Manipulación masiva de jobs y workers](/posts/resque-3-manipulacao-em-massa/)

> **Por qué existe esta serie** — trabajé mucho tiempo en proyectos de empresas que usaban Sidekiq y, durante ese proceso, fui compilando mis notas en la [serie de Sidekiq](/posts/sidekiq-1-infra-iniciando-parando-matando/). Después terminé entrando a un proyecto que usa Resque — eso generó nuevas notas, que mantuve con la misma estructura que las que ya había hecho para Sidekiq. Los problemas operacionales son los mismos (colas que se llenan, jobs que fallan en lote, deploy que necesita drenar workers), pero las herramientas cambian bastante. Esta serie es el espejo de aquella en los temas que se traducen bien: infra, diagnóstico y manipulación masiva, con los comandos equivalentes en Resque. Cuando algún tema tiene un paralelo directo, voy a enlazar con el post correspondiente de la serie Sidekiq.

La primera diferencia importante de modelo: **Resque es process-per-job, Sidekiq es thread-per-job**. Cada worker de Resque escucha una o más colas y, al tomar un job, hace `fork` de un proceso hijo que ejecuta el job y muere. Esto cambia la forma en que se inicia, drena y mata — porque siempre hay dos procesos relacionados (el worker padre y el hijo que está ejecutando).

Documentación de la API: [github.com/resque/resque](https://github.com/resque/resque).

## Iniciando Resque

```bash
QUEUE=* bundle exec rake resque:work
```

- `QUEUE=*` hace que el worker consuma todas las colas que existan en Redis.
- `QUEUE=critical,high,default` escucha en orden de prioridad — vacía `critical` antes de tocar `high`.

Levantando varios workers en una sola máquina:

```bash
COUNT=5 QUEUE=* bundle exec rake resque:workers
```

En segundo plano con PID file (útil para `kill` después):

```bash
PIDFILE=./tmp/pids/resque.pid BACKGROUND=yes QUEUE=* bundle exec rake resque:work
```

En producción esto vive en systemd o foreman, pero para VPS o ambiente de prueba el `BACKGROUND=yes + PIDFILE` es suficiente.

> A diferencia de Sidekiq, no existe `-C config/sidekiq.yml`. La concurrencia en Resque viene de levantar N procesos (no threads), entonces la "configuración" es cuántos `rake resque:work` tenés corriendo.

## Drenando con gracia (QUIT — recomendado)

```bash
kill -QUIT $(cat tmp/pids/resque.pid)
```

`QUIT` es la señal amable: el worker espera a que el job actual termine (en el proceso hijo), luego sale limpiamente. El equivalente moral de `sidekiqctl stop` de la [parte 1 de la serie Sidekiq](/posts/sidekiq-1-infra-iniciando-parando-matando/) — preserva idempotencia si el worker está en medio de una operación no atómica.

Si no tenés el PID file, podés extraerlo con `ps`:

```bash
ps -ef | grep '[r]esque' | grep -v 'master' | awk '{print $2}'
kill -QUIT $(ps -ef | grep '[r]esque' | grep -v 'master' | awk '{print $2}')
```

El `[r]esque` es el truco clásico para evitar que el propio `grep` aparezca en el resultado.

## Matando al hijo pero manteniendo el worker (TERM/USR1)

Como Resque hace fork por job, podés matar **solo el job en ejecución** sin derribar el worker padre:

```bash
# TERM en el worker: mata al hijo en el momento Y cierra el worker
kill -TERM $(cat tmp/pids/resque.pid)

# USR1 en el worker: mata al hijo en el momento, pero el worker sigue y toma el próximo job
kill -USR1 $(cat tmp/pids/resque.pid)
```

`USR1` es la herramienta correcta para "este job específico se trabó y está bloqueando el worker, pero no quiero tirar abajo la infraestructura". El job se convierte en failure, el worker vuelve al loop.

## Pausando sin matar (USR2 / CONT)

```bash
# USR2: deja de tomar jobs nuevos (no termina el actual, solo no toma el próximo)
kill -USR2 $(cat tmp/pids/resque.pid)

# CONT: vuelve a tomar jobs
kill -CONT $(cat tmp/pids/resque.pid)
```

Este par es el equivalente Resque del `Sidekiq::ProcessSet#quiet!` que aparece en la [parte 3 de la serie Sidekiq](/posts/sidekiq-3-manipulacao-em-massa/). Útil para deploy: `USR2` en todos los workers, esperás que los jobs en curso terminen, hacés el deploy, `CONT` para reanudar.

## Último recurso (KILL)

```bash
kill -9 $(cat tmp/pids/resque.pid)
```

`KILL` (`-9`) mata el worker y deja huérfano a cualquier hijo en ejecución — esos hijos se convierten en procesos zombie hasta que el init los recolecte, y el job en ejecución desaparece sin convertirse en failure (porque el worker no tuvo tiempo de registrarlo). Usá solo cuando:

- `QUIT` se trabó y no responde
- El worker está consumiendo memoria sin hacer nada visible
- Hay un incendio y derribarlo es más importante que preservar el estado

## Resumen de señales

| Señal | Efecto |
|-------|--------|
| `QUIT` | Termina los jobs en curso, luego sale (graceful) |
| `TERM` | Mata al hijo en el momento y sale |
| `USR1` | Mata al hijo en el momento, mantiene el worker corriendo |
| `USR2` | Deja de tomar jobs nuevos (pause) |
| `CONT` | Vuelve a tomar jobs (resume) |
| `KILL` (-9) | Mata el worker, deja hijos huérfanos |

## Próximo en la serie

[Diagnóstico de Resque por la consola](/posts/resque-2-diagnostico-pelo-console/) — `Resque.info`, conteos de colas, listado de workers y qué está haciendo cada uno.
