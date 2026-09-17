---
title: "Exponiendo tu Rails local con ngrok"
date: 2026-04-03 14:00:00 -0300
updated: 2026-04-03 14:00:00 -0300
tags: [rails, ngrok, webhooks, dev]
excerpt: "Setup de 2 minutos para dejar tu rails server accesible desde internet — útil para recibir webhooks de Stripe, Twilio o testear OAuth con un provider externo."
lang: es
ref: expondo-rails-local-com-ngrok
---

Los webhooks solo funcionan en un servidor público. Pero testear webhooks en un entorno de staging es tedioso — ngrok resuelve esto en segundos: crea un túnel HTTPS hacia tu puerto local.

## Pasos

```bash
brew install ngrok/ngrok/ngrok
```

Obtené el token en [dashboard.ngrok.com/get-started/setup](https://dashboard.ngrok.com/get-started/setup) y configuralo:

```bash
ngrok config add-authtoken TOKEN
```

Levantá el túnel apuntando al puerto de tu `rails server`:

```bash
ngrok http 3000
```

Copiá la URL `https://<algo>.ngrok-free.app` que aparece y pegala en el panel del provider que te va a llamar (Stripe, Twilio, GitHub, etc).

## Consideraciones

- **Hosts**: Rails 6+ bloquea hosts que no están en `config.hosts`. Agregá algo como `config.hosts << /.*\.ngrok-free\.app/` en `config/environments/development.rb`.
- **HTTPS**: La URL pública es `https`, pero la app local corre en `http`. Si usás `force_ssl`, desactivalo en dev.
- **La URL cambia**: Cada vez que reiniciás `ngrok` el subdominio cambia (en el plan free). Con cuenta paga podés reservar un subdominio fijo.
