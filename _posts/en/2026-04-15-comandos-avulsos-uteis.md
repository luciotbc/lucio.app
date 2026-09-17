---
title: "Miscellaneous commands I always forget"
date: 2026-04-15 11:00:00 -0300
updated: 2026-04-15 11:00:00 -0300
tags: [linux, ffmpeg, bash, dicas]
excerpt: "Three commands I use sporadically and always need to look up: see who's using a port, cut a video with ffmpeg, and identify the distro from a script."
lang: en
ref: comandos-avulsos-uteis
---

Some commands I use so rarely that every time I need them I have to go look them up again. Noting them here so I don't lose them.

## See who's using a port

```bash
lsof -n -i4TCP:8081
```

Replace `8081` with the port you want to investigate. `-n` avoids hostname resolution (faster). The output shows the process, PID, and user listening on that port.

## Cut a video segment with ffmpeg

```bash
ffmpeg -i video_original.mp4 -ss 00:58:00 -t 00:00:15 -c copy video_cortado.mp4
```

- `-ss 00:58:00` → starts at 58 minutes
- `-t 00:00:15` → duration of 15 seconds
- `-c copy` → copies the streams without re-encoding (fast and lossless)

Useful for extracting clips or creating samples from long videos.

## Identify the distro from a script

When you're in a script and need to know which distro you're running on:

```bash
awk -F= '$1=="ID" { print $2 ;}' /etc/os-release
```

Returns something like `arch`, `ubuntu`, `debian`, `manjaro`. You can use it in conditionals:

```bash
DISTRO=$(awk -F= '$1=="ID" { print $2 ;}' /etc/os-release)

if [ "$DISTRO" = "arch" ]; then
    pacman -S --needed pacote
elif [ "$DISTRO" = "ubuntu" ]; then
    apt install -y pacote
fi
```

## Bonus: close the lid without suspending

If you use your laptop as a server and don't want it to suspend when you close the lid:

```bash
sudo nvim /etc/systemd/logind.conf
# add:
# HandleLidSwitch=ignore

sudo nvim /etc/UPower/UPower.conf
# add:
# IgnoreLid=true
```

Useful for leaving the laptop processing something with the lid closed.
