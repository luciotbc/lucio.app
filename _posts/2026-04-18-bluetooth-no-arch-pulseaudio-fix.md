---
title: "Áudio Bluetooth no Arch/Manjaro: instalando o pulseaudio-bluetooth"
date: 2026-04-18 12:30:00 -0300
updated: 2026-04-18 12:30:00 -0300
tags: [linux, bluetooth, arch]
excerpt: "O Bluetooth conecta mas não toca som? Provavelmente está faltando o módulo do PulseAudio para Bluetooth."
---

O Bluetooth conecta mas o áudio não funciona, ou o dispositivo nem aparece nas opções de som? Provavelmente falta o módulo PulseAudio para Bluetooth:

```bash
sudo pacman -Sy pulseaudio-bluetooth --needed --noconfirm
```

Depois, habilite e inicie o serviço:

```bash
systemctl enable bluetooth.service
systemctl start bluetooth.service
```

O `--needed` evita reinstalar se já estiver instalado. Após isso, reinicie o PulseAudio (`killall pulseaudio` ou faça logout/login) e o dispositivo Bluetooth deve aparecer normalmente nas opções de áudio.
