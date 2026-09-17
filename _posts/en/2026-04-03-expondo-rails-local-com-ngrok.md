---
title: "Exposing your local Rails app with ngrok"
date: 2026-04-03 14:00:00 -0300
updated: 2026-04-03 14:00:00 -0300
tags: [rails, ngrok, webhooks, dev]
excerpt: "A 2-minute setup to make your rails server accessible from the internet — useful for receiving webhooks from Stripe, Twilio, or testing OAuth with an external provider."
lang: en
ref: expondo-rails-local-com-ngrok
---

Webhooks only work on a public server. But testing webhooks in a staging environment is tedious — ngrok solves this in seconds: it creates an HTTPS tunnel to your local port.

## Steps

```bash
brew install ngrok/ngrok/ngrok
```

Grab your token at [dashboard.ngrok.com/get-started/setup](https://dashboard.ngrok.com/get-started/setup) and configure it:

```bash
ngrok config add-authtoken TOKEN
```

Start the tunnel pointing to your `rails server` port:

```bash
ngrok http 3000
```

Copy the `https://<something>.ngrok-free.app` URL that appears and paste it in the provider's dashboard that will call you (Stripe, Twilio, GitHub, etc).

## Caveats

- **Hosts**: Rails 6+ blocks hosts not listed in `config.hosts`. Add something like `config.hosts << /.*\.ngrok-free\.app/` in `config/environments/development.rb`.
- **HTTPS**: The public URL is `https`, but the local app runs on `http`. If you use `force_ssl`, disable it in dev.
- **URL changes**: Every time `ngrok` restarts the subdomain changes (on the free plan). Paid accounts allow reserving a fixed subdomain.
