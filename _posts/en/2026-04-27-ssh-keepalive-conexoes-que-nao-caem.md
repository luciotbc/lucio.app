---
title: "SSH: configuring keepalive so connections don't drop from inactivity"
date: 2026-04-27 12:30:00 -0300
updated: 2026-04-27 12:30:00 -0300
tags: [linux, ssh]
excerpt: "How to prevent SSH connections from dropping due to inactivity by configuring keepalive on the server — or on the client, when you don't control the server."
lang: en
ref: ssh-keepalive-conexoes-que-nao-caem
---

SSH connections drop due to inactivity when there's no packet exchange for a period of time. To prevent this, configure keepalive on the server:

```bash
sudo vi /etc/ssh/sshd_config
```

Add or edit the lines:

```
TCPKeepAlive yes
ClientAliveInterval 30
ClientAliveCountMax 240
```

- `TCPKeepAlive yes` → enables keepalive at the TCP level
- `ClientAliveInterval 30` → sends a keepalive packet every 30 seconds
- `ClientAliveCountMax 240` → waits up to 240 attempts before closing (240 × 30s = 2 hours)

Restart the SSH service to apply:

```bash
sudo systemctl restart sshd
```

**Client-side alternative:** if you don't control the server, you can configure keepalive in your `~/.ssh/config`:

```
Host *
    ServerAliveInterval 30
    ServerAliveCountMax 240
```

Works the same way, but configured from the client side.
