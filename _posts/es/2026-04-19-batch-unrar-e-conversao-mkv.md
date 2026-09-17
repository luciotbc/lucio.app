---
title: "Procesamiento de archivos en lote: extraer .rar y convertir MKV en batch"
date: 2026-04-19 13:00:00 -0300
updated: 2026-04-19 13:00:00 -0300
tags: [linux, bash, ffmpeg, batch, automação]
excerpt: "Dos loops bash para procesar archivos en lote: extraer todos los .rar de un directorio y convertir todos los .mkv a .mp4 con ffmpeg."
lang: es
ref: batch-unrar-e-conversao-mkv
---

Cuando tenés una carpeta llena de archivos para procesar, un loop bash lo resuelve sin necesidad de abrir ninguna GUI.

## Extraer todos los .rar de un directorio

```bash
for f in *.rar; do unrar e "$f"; done
```

`unrar e` extrae los archivos sin preservar la estructura de directorios (todo va a la carpeta actual). Si querés preservar la estructura de carpetas, usá `unrar x` en lugar de `unrar e`.

Si los `.rar` son partes de un único archivo (`.part1.rar`, `.part2.rar`, etc.), basta con ejecutarlo en el primero:

```bash
unrar e arquivo.part1.rar
```

`unrar` encuentra las otras partes automáticamente.

## Convertir todos los .mkv a .mp4

```bash
for f in *.mkv; do ffmpeg -hide_banner -i "$f" -c:v libx264 -c:a copy "$f.mp4"; done
```

- `-hide_banner` → suprime el largo encabezado de ffmpeg
- `-c:v libx264` → re-encoda el video en H.264 (compatible con prácticamente todo)
- `-c:a copy` → copia el audio sin re-encodar (más rápido y sin pérdida)
- `"$f.mp4"` → el nombre de salida queda `video.mkv.mp4` — si querés `video.mp4`, usá `"${f%.mkv}.mp4"`

Versión con nombre limpio:

```bash
for f in *.mkv; do ffmpeg -hide_banner -i "$f" -c:v libx264 -c:a copy "${f%.mkv}.mp4"; done
```

## Consideraciones de performance

La conversión de video es intensiva en CPU. Si tenés una GPU NVIDIA y ffmpeg compilado con soporte a NVENC, podés acelerar mucho:

```bash
ffmpeg -i "$f" -c:v h264_nvenc -c:a copy "${f%.mkv}.mp4"
```

Para Intel con Quick Sync:

```bash
ffmpeg -i "$f" -c:v h264_qsv -c:a copy "${f%.mkv}.mp4"
```

Pero para uso general, `libx264` con `-preset fast` ya es un buen equilibrio:

```bash
ffmpeg -i "$f" -c:v libx264 -preset fast -c:a copy "${f%.mkv}.mp4"
```
