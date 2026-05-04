---
title: "Bootstrap de SaaS em Rails: o template que eu uso"
date: 2026-04-06 14:00:00 -0300
updated: 2026-04-06 14:00:00 -0300
published: false # só publico depois de ter os próximos passos prontos
tags: [ruby, rails, setup, graphql, devise]
excerpt: "Comando rails new que eu uso pra começar projeto novo, e os ajustes pra deixar GraphQL com GraphiQL funcionando junto com Sprockets."
---

Toda vez que vou começar um SaaS novo em Rails, eu acabo passando uns 30 minutos lembrando das mesmas flags. Esse post é meu cheat sheet.

## A evolução do meu `rails new`

```bash
# v1
rails new app --database=postgresql

# v2
rails new app --database=postgresql --webpack=react

# v3 (API only)
rails new api --database=postgresql --skip-javascript --skip-turbolinks --skip-sprockets --skip-action-text --skip-webpack-install

# v4 (atual)
rails new app -d postgresql \
  --skip-action-mailbox \
  --skip-action-text \
  --skip-spring \
  --webpack=react \
  -T \
  --skip-turbolinks
```

A v4 é o meu padrão hoje: PostgreSQL, React via Webpacker, sem Action Text/Mailbox/Spring/Turbolinks, sem framework de teste padrão (`-T`) — eu prefiro setar RSpec depois.

## Setup de banco

`rails db:create` resolve, mas vale conferir `config/database.yml` pra ajustar host/usuário antes de qualquer coisa.

## Setup de GraphQL com GraphiQL

```bash
rails generate graphql:install --relay --batch
```

Pra que o GraphiQL funcione junto com a stack moderna do Rails (sem Sprockets por padrão), tem dois ajustes:

Em `config/application.rb`, descomente/adicione:

```ruby
require "sprockets/railtie"
```

E em `app/assets/config/manifest.js`, adicione:

```js
//= link graphiql/rails/application.css
//= link graphiql/rails/application.js
```

Sem isso, a interface do GraphiQL fica sem CSS/JS e vira só um formulário pelado.

## Próximos passos do template

- Setup do Devise pra autenticação
- Setup do Docker (`Dockerfile` + `docker-compose.yml` pra Rails + Postgres + Redis)
- Fluxo de login funcionando ponta a ponta

Esses três eu trato em posts separados — cada um vira um pequeno tutorial.
