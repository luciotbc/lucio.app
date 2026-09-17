---
title: "Bluetooth audio on Arch/Manjaro: installing pulseaudio-bluetooth"
date: 2026-04-18 12:30:00 -0300
updated: 2026-04-18 12:30:00 -0300
tags: [linux, bluetooth, arch]
excerpt: "Bluetooth connects but no sound? You're probably missing the PulseAudio module for Bluetooth."
lang: en
ref: bluetooth-no-arch-pulseaudio-fix
---

Bluetooth connects but audio doesn't work, or the device doesn't even show up in the sound options? You're probably missing the PulseAudio module for Bluetooth:

```bash
sudo pacman -Sy pulseaudio-bluetooth --needed --noconfirm
```

Then, enable and start the service:

```bash
systemctl enable bluetooth.service
systemctl start bluetooth.service
```

`--needed` avoids reinstalling if it's already installed. After that, restart PulseAudio (`killall pulseaudio` or log out/log in) and the Bluetooth device should appear normally in the audio options.
