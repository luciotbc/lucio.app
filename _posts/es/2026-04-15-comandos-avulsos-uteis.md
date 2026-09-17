---
title: "Comandos sueltos que siempre olvido"
date: 2026-04-15 11:00:00 -0300
updated: 2026-04-15 11:00:00 -0300
tags: [linux, ffmpeg, bash, dicas]
excerpt: "Tres comandos que uso esporádicamente y siempre necesito buscar: ver quién usa un puerto, cortar un video con ffmpeg e identificar la distro desde un script."
lang: es
ref: comandos-avulsos-uteis
---

Algunos comandos los uso tan raramente que cada vez que los necesito tengo que ir a buscarlos de nuevo. Los anoto acá para no perderlos.

## Ver quién está usando un puerto

```bash
lsof -n -i4TCP:8081
```

Reemplazá `8081` por el puerto que querés investigar. `-n` evita la resolución de hostname (más rápido). La salida muestra el proceso, PID y usuario que está escuchando en ese puerto.

## Cortar un segmento de video con ffmpeg

```bash
ffmpeg -i video_original.mp4 -ss 00:58:00 -t 00:00:15 -c copy video_cortado.mp4
```

- `-ss 00:58:00` → comienza en el minuto 58
- `-t 00:00:15` → duración de 15 segundos
- `-c copy` → copia los streams sin re-encodear (rápido y sin pérdida de calidad)

Útil para extraer clips o crear muestras de videos largos.

## Identificar la distro desde un script

Cuando estás en un script y necesitás saber en qué distro estás corriendo:

```bash
awk -F= '$1=="ID" { print $2 ;}' /etc/os-release
```

Devuelve algo como `arch`, `ubuntu`, `debian`, `manjaro`. Se puede usar en condicionales:

```bash
DISTRO=$(awk -F= '$1=="ID" { print $2 ;}' /etc/os-release)

if [ "$DISTRO" = "arch" ]; then
    pacman -S --needed pacote
elif [ "$DISTRO" = "ubuntu" ]; then
    apt install -y pacote
fi
```

## Bonus: cerrar la tapa sin suspender

Si usás la notebook como servidor y no querés que se suspenda al cerrar la tapa:

```bash
sudo nvim /etc/systemd/logind.conf
# agregar:
# HandleLidSwitch=ignore

sudo nvim /etc/UPower/UPower.conf
# agregar:
# IgnoreLid=true
```

Útil para dejar la notebook procesando algo con la tapa cerrada.
