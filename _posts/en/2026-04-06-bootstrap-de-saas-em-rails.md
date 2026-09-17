---
title: "SaaS bootstrap in Rails: the template I use"
date: 2026-04-06 14:00:00 -0300
updated: 2026-04-06 14:00:00 -0300
tags: [ruby, rails, setup, graphql, devise]
excerpt: "The rails new command I use to start a new project, and the adjustments to get GraphQL with GraphiQL working alongside Sprockets."
lang: en
ref: bootstrap-de-saas-em-rails
---

Every time I start a new SaaS in Rails, I end up spending 30 minutes remembering the same flags. This post is my cheat sheet.

## The evolution of my `rails new`

```bash
# v1
rails new app --database=postgresql

# v2
rails new app --database=postgresql --webpack=react

# v3 (API only)
rails new api --database=postgresql --skip-javascript --skip-turbolinks --skip-sprockets --skip-action-text --skip-webpack-install

# v4 (current)
rails new app -d postgresql \
  --skip-action-mailbox \
  --skip-action-text \
  --skip-spring \
  --webpack=react \
  -T \
  --skip-turbolinks
```

v4 is my current default: PostgreSQL, React via Webpacker, without Action Text/Mailbox/Spring/Turbolinks, without a default test framework (`-T`) — I prefer setting up RSpec afterward.

## Database setup

`rails db:create` does the job, but it's worth checking `config/database.yml` to adjust host/user before anything else.

## GraphQL with GraphiQL setup

```bash
rails generate graphql:install --relay --batch
```

For GraphiQL to work with the modern Rails stack (without Sprockets by default), there are two adjustments:

In `config/application.rb`, uncomment/add:

```ruby
require "sprockets/railtie"
```

And in `app/assets/config/manifest.js`, add:

```js
//= link graphiql/rails/application.css
//= link graphiql/rails/application.js
```

Without this, the GraphiQL interface has no CSS/JS and becomes just a bare form.

## Next steps in the template

- Devise setup for authentication
- Docker setup (`Dockerfile` + `docker-compose.yml` for Rails + Postgres + Redis)
- Working login flow end to end

Those three I cover in separate posts — each one becomes a small tutorial.
