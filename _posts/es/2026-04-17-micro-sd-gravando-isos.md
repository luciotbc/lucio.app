---
title: "Micro SD: listar discos, grabar ISOs y borrar de forma segura"
date: 2026-04-17 12:00:00 -0300
updated: 2026-04-17 12:00:00 -0300
tags: [linux, dd, micro-sd, raspberry-pi, iso]
excerpt: "Cómo identificar la tarjeta SD en el sistema, grabar una ISO directamente con dd o xzcat, y borrar el disco de forma segura con shred."
lang: es
ref: micro-sd-gravando-isos
---

Grabar una imagen de sistema en una tarjeta SD o pendrive desde la terminal es más simple de lo que parece — y más rápido que abrir Etcher la mayoría de las veces.

## Identificar el disco correcto

Antes que nada, identificar el dispositivo correcto es crítico. `dd` no perdona errores:

```bash
lsblk -p
```

El `-p` muestra la ruta completa del dispositivo (ej: `/dev/sda`, `/dev/mmcblk0`). Busca el tamaño correspondiente a tu tarjeta SD.

## Grabar una ISO directamente

```bash
sudo dd if=linux-2020.4-live-amd64.iso of=/dev/sda bs=4M conv=fsync status=progress && sync
```

- `if=` → archivo de entrada (la ISO)
- `of=` → dispositivo de salida (la tarjeta SD — **sin número de partición**: `/dev/sda`, no `/dev/sda1`)
- `bs=4M` → tamaño del bloque de transferencia (4MB es un buen equilibrio)
- `conv=fsync` → garantiza que los datos sean escritos físicamente
- `status=progress` → muestra el progreso en tiempo real
- `&& sync` → fuerza la sincronización antes de finalizar

## Grabar una ISO comprimida en .xz

Las imágenes de Raspberry Pi generalmente vienen comprimidas en `.xz`. Se puede descomprimir y grabar en un solo comando:

```bash
sudo xzcat linux-2020.4-live-amd64.iso.xz | dd of=/dev/sda bs=32M conv=fsync status=progress && sync
```

O de forma más genérica:

```bash
xzcat ~/Downloads/<archivo.xz> | sudo dd of=<dispositivo> bs=32M
```

Si preferís descomprimir primero:

```bash
xz -d server-arm64+raspi.img.xz
```

Esto genera el archivo `.img` que puede grabarse normalmente con `dd`.

## Borrar el disco de forma segura (zero fill)

Para borrar un disco antes de descartarlo o reutilizarlo:

```bash
sudo shred -n 2 -z -v /dev/sda1
```

- `-n 2` → hace 2 pasadas con datos aleatorios
- `-z` → hace una pasada final con ceros (disimula el shredding)
- `-v` → verbose, muestra el progreso

> **Atención:** para SSDs y flash storage (como tarjetas SD), `shred` no garantiza borrado completo por el wear leveling. Para esos casos, un único `dd if=/dev/zero` ya es suficiente para uso práctico.
