---
title: "PostgreSQL en Mac con Homebrew: instalación y bugs comunes"
date: 2026-04-20 14:00:00 -0300
updated: 2026-04-20 14:00:00 -0300
tags: [postgresql, mac, homebrew, setup, ops]
excerpt: "Levantar Postgres en Mac vía brew es fácil — hasta la primera vez que no arranca. Comandos para instalar, leer el log, eliminar el lock file y migrar el data directory después de un upgrade."
lang: es
ref: postgres-mac-homebrew-instalacao-e-bugs
---

> **Serie PostgreSQL** — parte 1 de 2
>
> 1. **Estás aquí — Instalación en Mac y bugs comunes**
> 2. [Tuning e investigación por la consola SQL](/posts/postgres-tuning-e-investigacao-pelo-console/)

Levantar Postgres en Mac vía Homebrew es trivial cuando todo está bien. El problema es que, cuando algo se traba, el mensaje de error raramente dice qué hacer. Este post es la secuencia de comandos que ejecuto, en orden, cuando `brew services start` falla — más lo que hago después de un upgrade de versión mayor.

## Instalando

```bash
brew install postgresql@14
```

Siempre fijo la versión (`@14`, `@15`, `@16`) en vez de usar la fórmula sin número. Las versiones mayores de Postgres cambian el formato de datos en disco, e instalar "la más nueva" sin darse cuenta es una receta para perder la base de datos.

## Iniciar, detener, reiniciar

```bash
brew services start postgresql@14
brew services stop postgresql@14
brew services restart postgresql@14
```

Sin secretos. El servicio sigue corriendo entre reinicios; para correr Postgres solo "ahora" usá `pg_ctl -D /opt/homebrew/var/postgresql@14 start` directamente.

## Cuando no arranca: leer el log

Si `brew services start` devuelve `error` o `psql` rechaza la conexión, antes que nada mirá el log:

```bash
brew services stop postgresql@14
rm /opt/homebrew/var/log/postgresql@14.log
brew services start postgresql@14
cat /opt/homebrew/var/log/postgresql@14.log
```

Borrar el log antes de arrancar limpia el ruido de sesiones anteriores — solo leés lo que pasó en ese intento.

## Error: "lock file postmaster.pid already exists"

```text
FATAL:  lock file "postmaster.pid" already exists
```

Ocurre cuando Postgres murió sin cerrarse correctamente (kernel panic, `kill -9`, batería agotada). El proceso ya no está, pero el archivo de lock quedó. Solución:

```bash
rm -rf /opt/homebrew/var/postgresql@14/postmaster.pid
brew services start postgresql@14
```

Si no estás seguro de que no hay ningún proceso Postgres corriendo, verificá antes con `ps -ef | grep postgres | grep -v grep`. Borrar el lock con el proceso vivo puede causar corrupción.

## Después de un `brew upgrade`: el data directory cambió de lugar

Cuando actualizás Postgres por una versión mayor, Homebrew cambia el directorio de datos. El log lo deja claro:

```text
You can migrate to a versioned data directory by running:
  mv -v "/opt/homebrew/var/postgres" "/opt/homebrew/var/postgresql@14"
```

Antes de mover, **detené el servicio**:

```bash
brew services stop postgresql@14
mv -v /opt/homebrew/var/postgres /opt/homebrew/var/postgresql@14
brew services start postgresql@14
```

Si ya iniciaste el servicio sin migrar y creó un cluster vacío en `postgresql@14`, podés terminar con dos data dirs — uno nuevo vacío y otro con tus datos. Ya perdí una base así. Si eso pasa: detené el servicio, borrá el cluster vacío y repetí el `mv` antes de arrancar.

Para ver detalles de la fórmula instalada:

```bash
brew info postgresql@14
```

## Crear usuarios por defecto

Una instalación nueva de Postgres en Mac no tiene el usuario `postgres` (por defecto el owner es tu usuario del sistema). Para tener lo que otras herramientas esperan:

```bash
createuser -s postgres
createuser --interactive --pwprompt
createdb init_test
```

- `createuser -s postgres` crea el superuser `postgres` (sin contraseña). Útil para herramientas que asumen ese usuario.
- `createuser --interactive --pwprompt` hace un wizard para crear un usuario común con contraseña.
- `createdb init_test` crea una base de datos de prueba solo para confirmar que todo funciona.

## Habilitar `pg_stat_statements`

`pg_stat_statements` es la extensión más útil para quien quiere entender consultas lentas. Viene incluida en la instalación, pero necesita ser cargada al arrancar.

Editá `/opt/homebrew/var/postgresql@14/postgresql.conf` y agregá:

```ini
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.track = all
```

O desde la línea de comandos:

```bash
echo "shared_preload_libraries = 'pg_stat_statements'" >> /opt/homebrew/var/postgresql@14/postgresql.conf
echo "pg_stat_statements.track = all" >> /opt/homebrew/var/postgresql@14/postgresql.conf
brew services restart postgresql@14
```

Reiniciá el servicio (un reload no alcanza — `shared_preload_libraries` solo se lee al arrancar) y creá la extensión en la base de datos donde querés medir:

```sql
CREATE EXTENSION pg_stat_statements;
SELECT * FROM pg_stat_statements LIMIT 5;
```

Si la query de arriba devuelve filas, está funcionando. Cómo usar esto para cazar queries lentas queda para el [próximo post de la serie](/posts/postgres-tuning-e-investigacao-pelo-console/).

Referencia sobre cómo habilitarlo con más profundidad: [bytebase.com/docs/slow-query/enable-pg-stat-statements-for-postgresql](https://www.bytebase.com/docs/slow-query/enable-pg-stat-statements-for-postgresql/).

## Próximo en la serie

[Tuning e investigación por la consola SQL](/posts/postgres-tuning-e-investigacao-pelo-console/) — `EXPLAIN ANALYZE` en PEV2, ajustar `work_mem`, listar uso de índices, contar filas de todas las tablas y más.
