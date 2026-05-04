---
title: "Batch de arquivos: extrair .rar e converter MKV em lote"
date: 2026-04-19 13:00:00 -0300
updated: 2026-04-19 13:00:00 -0300
tags: [linux, bash, ffmpeg, batch, automação]
excerpt: "Dois loops bash pra processar arquivos em lote: extrair todos os .rar de um diretório e converter todos os .mkv para .mp4 com ffmpeg."
---

Quando você tem uma pasta cheia de arquivos pra processar, um loop bash resolve sem precisar abrir nenhuma GUI.

## Extrair todos os .rar de um diretório

```bash
for f in *.rar; do unrar e "$f"; done
```

O `unrar e` extrai os arquivos sem preservar a estrutura de diretórios (tudo vai para a pasta atual). Se quiser preservar a estrutura de pastas, use `unrar x` em vez de `unrar e`.

Se os `.rar` são partes de um único arquivo (`.part1.rar`, `.part2.rar`, etc.), basta rodar no primeiro:

```bash
unrar e arquivo.part1.rar
```

O `unrar` encontra as outras partes automaticamente.

## Converter todos os .mkv para .mp4

```bash
for f in *.mkv; do ffmpeg -hide_banner -i "$f" -c:v libx264 -c:a copy "$f.mp4"; done
```

- `-hide_banner` → suprime o cabeçalho longo do ffmpeg
- `-c:v libx264` → re-encoda o vídeo em H.264 (compatível com praticamente tudo)
- `-c:a copy` → copia o áudio sem re-encodar (mais rápido e sem perda)
- `"$f.mp4"` → o nome de saída fica `video.mkv.mp4` — se quiser `video.mp4`, use `"${f%.mkv}.mp4"`

Versão com o nome limpo:

```bash
for f in *.mkv; do ffmpeg -hide_banner -i "$f" -c:v libx264 -c:a copy "${f%.mkv}.mp4"; done
```

## Considerações de performance

A conversão de vídeo é CPU-intensiva. Se tiver uma GPU NVIDIA e o ffmpeg compilado com suporte a NVENC, dá pra acelerar muito:

```bash
ffmpeg -i "$f" -c:v h264_nvenc -c:a copy "${f%.mkv}.mp4"
```

Para Intel com Quick Sync:

```bash
ffmpeg -i "$f" -c:v h264_qsv -c:a copy "${f%.mkv}.mp4"
```

Mas para uso geral, o `libx264` com `-preset fast` já é um bom equilíbrio:

```bash
ffmpeg -i "$f" -c:v libx264 -preset fast -c:a copy "${f%.mkv}.mp4"
```
