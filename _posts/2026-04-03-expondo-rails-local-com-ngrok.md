---
title: "Expondo o Rails local com ngrok"
date: 2026-04-03 14:00:00 -0300
updated: 2026-04-03 14:00:00 -0300
tags: [rails, ngrok, webhooks, dev]
excerpt: "Setup de 2 minutos pra deixar seu rails server acessível pela internet — útil pra receber webhook de Stripe, Twilio ou testar OAuth de provider externo."
---

Webhook só funciona em servidor público. Mas testar webhook em ambiente de homolog é chato — o ngrok resolve isso em segundos: ele cria um túnel HTTPS pra sua porta local.

## Passos

```bash
brew install ngrok/ngrok/ngrok
```

Pega o token em [dashboard.ngrok.com/get-started/setup](https://dashboard.ngrok.com/get-started/setup) e configura:

```bash
ngrok config add-authtoken TOKEN
```

Sobe o túnel apontando pra porta do `rails server`:

```bash
ngrok http 3000
```

Copia a URL `https://<algo>.ngrok-free.app` que aparece e cola no painel do provider que vai te chamar (Stripe, Twilio, GitHub, etc).

## Cuidados

- **Hosts**: o Rails 6+ bloqueia hosts que não estão em `config.hosts`. Adicione algo como `config.hosts << /.*\.ngrok-free\.app/` em `config/environments/development.rb`.
- **HTTPS**: a URL pública é `https`, mas o app local roda `http`. Se você usa `force_ssl`, desligue em dev.
- **URL muda**: a cada reinício do `ngrok` a subdomínio muda (no plano free). Em conta paga dá pra reservar um subdomínio fixo.
