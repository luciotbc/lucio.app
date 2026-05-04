---
title: "PostgreSQL no Mac com Homebrew: instalação e bugs comuns"
date: 2026-04-20 14:00:00 -0300
updated: 2026-04-20 14:00:00 -0300
tags: [postgresql, mac, homebrew, setup, ops]
excerpt: "Subir o Postgres no Mac via brew é fácil — até a primeira vez que ele não sobe. Comandos pra instalar, ler o log, destravar lock file e migrar o data directory depois de upgrade."
---

> **Série PostgreSQL** — parte 1 de 2
>
> 1. **Você está aqui — Instalação no Mac e bugs comuns**
> 2. [Tuning e investigação pelo console SQL]({% post_url 2026-04-21-postgres-tuning-e-investigacao-pelo-console %})

Subir Postgres no Mac via Homebrew é trivial enquanto está tudo bem. O problema é que, quando algo trava, a mensagem de erro raramente diz o que fazer. Esse post é a sequência de comandos que eu rodo, na ordem, quando o `brew services start` falha — mais o que faço depois de um upgrade de versão maior.

## Instalando

```bash
brew install postgresql@14
```

Eu sempre fixo a versão (`@14`, `@15`, `@16`) em vez de usar a fórmula sem número. Major version do Postgres mexe com formato de dados em disco, e instalar "a mais nova" sem perceber é receita pra perder banco.

## Subindo, parando, reiniciando

```bash
brew services start postgresql@14
brew services stop postgresql@14
brew services restart postgresql@14
```

Sem segredo. O serviço fica rodando entre reboots; pra rodar Postgres só "agora" use `pg_ctl -D /opt/homebrew/var/postgresql@14 start` direto.

## Quando ele não sobe: ler o log

Se `brew services start` retorna `error` ou o `psql` recusa conexão, antes de qualquer coisa olhe o log:

```bash
brew services stop postgresql@14
rm /opt/homebrew/var/log/postgresql@14.log
brew services start postgresql@14
cat /opt/homebrew/var/log/postgresql@14.log
```

Apagar o log antes de subir limpa o ruído de sessões anteriores — você lê só o que aconteceu nessa tentativa.

## Erro: "lock file postmaster.pid already exists"

```text
FATAL:  lock file "postmaster.pid" already exists
```

Acontece quando o Postgres morreu sem fechar direito (kernel panic, `kill -9`, bateria que acabou). O processo não está mais lá, mas o arquivo de lock ficou. Solução:

```bash
rm -rf /opt/homebrew/var/postgresql@14/postmaster.pid
brew services start postgresql@14
```

Se você não tem certeza de que nenhum processo Postgres está rodando, confira antes com `ps -ef | grep postgres | grep -v grep`. Apagar o lock com o processo vivo é capaz de causar corrupção.

## Depois de um `brew upgrade`: data directory mudou de lugar

Quando você atualiza o Postgres por uma major version, o Homebrew muda o diretório de dados. O log fica claro:

```text
You can migrate to a versioned data directory by running:
  mv -v "/opt/homebrew/var/postgres" "/opt/homebrew/var/postgresql@14"
```

Antes de mover **pare o serviço**:

```bash
brew services stop postgresql@14
mv -v /opt/homebrew/var/postgres /opt/homebrew/var/postgresql@14
brew services start postgresql@14
```

Se você já abriu o serviço sem migrar e ele criou um cluster vazio em `postgresql@14`, dá pra acabar com dois data dirs — um vazio novo e outro com seus dados. Já perdi base assim. Se isso acontecer: pare o serviço, apague o cluster vazio e refaça o `mv` antes de subir.

Pra ver detalhes da fórmula instalada:

```bash
brew info postgresql@14
```

## Criando usuários default

Postgres novo não tem o usuário `postgres` no Mac (por padrão o owner é seu usuário do sistema). Pra ter o que outras ferramentas esperam:

```bash
createuser -s postgres
createuser --interactive --pwprompt
createdb init_test
```

- `createuser -s postgres` cria o superuser `postgres` (sem senha). Útil pra ferramentas que assumem esse user.
- `createuser --interactive --pwprompt` faz wizard pra criar um usuário comum com senha.
- `createdb init_test` cria um banco de teste só pra confirmar que tudo está funcionando.

## Habilitando `pg_stat_statements`

`pg_stat_statements` é a extensão mais útil de quem quer entender consultas lentas. Ela vem junto da instalação, mas precisa ser carregada no boot.

Edite `/opt/homebrew/var/postgresql@14/postgresql.conf` e adicione:

```ini
shared_preload_libraries = 'pg_stat_statements'
pg_stat_statements.track = all
```

Ou, na linha de comando:

```bash
echo "shared_preload_libraries = 'pg_stat_statements'" >> /opt/homebrew/var/postgresql@14/postgresql.conf
echo "pg_stat_statements.track = all" >> /opt/homebrew/var/postgresql@14/postgresql.conf
brew services restart postgresql@14
```

Reinicie o serviço (não basta um reload — `shared_preload_libraries` só lê no startup) e crie a extensão num banco onde você quer medir:

```sql
CREATE EXTENSION pg_stat_statements;
SELECT * FROM pg_stat_statements LIMIT 5;
```

Se a query de cima retorna linhas, está funcionando. Como usar isso pra caçar query lenta fica pro [próximo post da série]({% post_url 2026-04-21-postgres-tuning-e-investigacao-pelo-console %}).

Referência de como ativar com mais profundidade: [bytebase.com/docs/slow-query/enable-pg-stat-statements-for-postgresql](https://www.bytebase.com/docs/slow-query/enable-pg-stat-statements-for-postgresql/).

## Próximo da série

[Tuning e investigação pelo console SQL]({% post_url 2026-04-21-postgres-tuning-e-investigacao-pelo-console %}) — `EXPLAIN ANALYZE` no PEV2, ajustar `work_mem`, listar uso de índice, contar linhas de todas as tabelas e mais.
