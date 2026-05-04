---
title: "SSH: configurando keepalive para a conexão não cair por ociosidade"
date: 2026-04-27 12:30:00 -0300
updated: 2026-04-27 12:30:00 -0300
tags: [linux, ssh]
excerpt: "Como evitar que conexões SSH caiam por inatividade configurando keepalive no servidor — ou no cliente, quando você não controla o servidor."
---

Conexões SSH caem por inatividade quando não há troca de pacotes por um tempo. Pra evitar isso, configure o keepalive no servidor:

```bash
sudo vi /etc/ssh/sshd_config
```

Adicione ou edite as linhas:

```
TCPKeepAlive yes
ClientAliveInterval 30
ClientAliveCountMax 240
```

- `TCPKeepAlive yes` → ativa o keepalive a nível TCP
- `ClientAliveInterval 30` → envia um pacote de keepalive a cada 30 segundos
- `ClientAliveCountMax 240` → aguarda até 240 tentativas antes de encerrar (240 × 30s = 2 horas)

Reinicie o serviço SSH para aplicar:

```bash
sudo systemctl restart sshd
```

**Alternativa no cliente:** se você não controla o servidor, pode configurar o keepalive no seu `~/.ssh/config`:

```
Host *
    ServerAliveInterval 30
    ServerAliveCountMax 240
```

Funciona da mesma forma, mas configurado pelo lado do cliente.
