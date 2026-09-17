---
title: "Bootstrap de SaaS en Rails: el template que uso"
date: 2026-04-06 14:00:00 -0300
updated: 2026-04-06 14:00:00 -0300
tags: [ruby, rails, setup, graphql, devise]
excerpt: "El comando rails new que uso para empezar un proyecto nuevo, y los ajustes para que GraphQL con GraphiQL funcione junto con Sprockets."
lang: es
ref: bootstrap-de-saas-em-rails
---

Cada vez que voy a empezar un SaaS nuevo en Rails, termino pasando unos 30 minutos recordando los mismos flags. Este post es mi cheat sheet.

## La evolución de mi `rails new`

```bash
# v1
rails new app --database=postgresql

# v2
rails new app --database=postgresql --webpack=react

# v3 (API only)
rails new api --database=postgresql --skip-javascript --skip-turbolinks --skip-sprockets --skip-action-text --skip-webpack-install

# v4 (actual)
rails new app -d postgresql \
  --skip-action-mailbox \
  --skip-action-text \
  --skip-spring \
  --webpack=react \
  -T \
  --skip-turbolinks
```

La v4 es mi estándar actual: PostgreSQL, React via Webpacker, sin Action Text/Mailbox/Spring/Turbolinks, sin framework de testing por defecto (`-T`) — prefiero configurar RSpec después.

## Setup de base de datos

`rails db:create` resuelve, pero vale revisar `config/database.yml` para ajustar host/usuario antes de cualquier cosa.

## Setup de GraphQL con GraphiQL

```bash
rails generate graphql:install --relay --batch
```

Para que GraphiQL funcione junto con el stack moderno de Rails (sin Sprockets por defecto), hay dos ajustes:

En `config/application.rb`, descomentá/agregá:

```ruby
require "sprockets/railtie"
```

Y en `app/assets/config/manifest.js`, agregá:

```js
//= link graphiql/rails/application.css
//= link graphiql/rails/application.js
```

Sin esto, la interfaz de GraphiQL queda sin CSS/JS y se convierte en un formulario pelado.

## Próximos pasos del template

- Setup de Devise para autenticación
- Setup de Docker (`Dockerfile` + `docker-compose.yml` para Rails + Postgres + Redis)
- Flujo de login funcionando de punta a punta

Esos tres los trato en posts separados — cada uno se convierte en un pequeño tutorial.
