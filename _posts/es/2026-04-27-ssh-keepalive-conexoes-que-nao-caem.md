---
title: "SSH: configurando keepalive para que la conexión no caiga por inactividad"
date: 2026-04-27 12:30:00 -0300
updated: 2026-04-27 12:30:00 -0300
tags: [linux, ssh]
excerpt: "Cómo evitar que las conexiones SSH caigan por inactividad configurando keepalive en el servidor — o en el cliente, cuando no controlás el servidor."
lang: es
ref: ssh-keepalive-conexoes-que-nao-caem
---

Las conexiones SSH caen por inactividad cuando no hay intercambio de paquetes por un tiempo. Para evitarlo, configurá el keepalive en el servidor:

```bash
sudo vi /etc/ssh/sshd_config
```

Agregá o editá las líneas:

```
TCPKeepAlive yes
ClientAliveInterval 30
ClientAliveCountMax 240
```

- `TCPKeepAlive yes` → activa el keepalive a nivel TCP
- `ClientAliveInterval 30` → envía un paquete de keepalive cada 30 segundos
- `ClientAliveCountMax 240` → espera hasta 240 intentos antes de cerrar (240 × 30s = 2 horas)

Reiniciá el servicio SSH para aplicar:

```bash
sudo systemctl restart sshd
```

**Alternativa en el cliente:** si no controlás el servidor, podés configurar el keepalive en tu `~/.ssh/config`:

```
Host *
    ServerAliveInterval 30
    ServerAliveCountMax 240
```

Funciona de la misma forma, pero configurado desde el lado del cliente.
