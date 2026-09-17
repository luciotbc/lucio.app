---
title: "Audio Bluetooth en Arch/Manjaro: instalando pulseaudio-bluetooth"
date: 2026-04-18 12:30:00 -0300
updated: 2026-04-18 12:30:00 -0300
tags: [linux, bluetooth, arch]
excerpt: "¿Bluetooth conecta pero no hay sonido? Probablemente falta el módulo de PulseAudio para Bluetooth."
lang: es
ref: bluetooth-no-arch-pulseaudio-fix
---

¿Bluetooth conecta pero el audio no funciona, o el dispositivo ni siquiera aparece en las opciones de sonido? Probablemente falta el módulo PulseAudio para Bluetooth:

```bash
sudo pacman -Sy pulseaudio-bluetooth --needed --noconfirm
```

Después, habilitá e iniciá el servicio:

```bash
systemctl enable bluetooth.service
systemctl start bluetooth.service
```

`--needed` evita reinstalar si ya está instalado. Luego de eso, reiniciá PulseAudio (`killall pulseaudio` o hacé logout/login) y el dispositivo Bluetooth debería aparecer normalmente en las opciones de audio.
