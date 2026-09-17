---
title: "Batch file processing: extract .rar and convert MKV in bulk"
date: 2026-04-19 13:00:00 -0300
updated: 2026-04-19 13:00:00 -0300
tags: [linux, bash, ffmpeg, batch, automação]
excerpt: "Two bash loops to process files in bulk: extract all .rar files from a directory and convert all .mkv to .mp4 with ffmpeg."
lang: en
ref: batch-unrar-e-conversao-mkv
---

When you have a folder full of files to process, a bash loop handles it without needing to open any GUI.

## Extract all .rar files from a directory

```bash
for f in *.rar; do unrar e "$f"; done
```

`unrar e` extracts files without preserving the directory structure (everything goes to the current folder). If you want to preserve the folder structure, use `unrar x` instead of `unrar e`.

If the `.rar` files are parts of a single archive (`.part1.rar`, `.part2.rar`, etc.), just run it on the first one:

```bash
unrar e arquivo.part1.rar
```

`unrar` finds the other parts automatically.

## Convert all .mkv to .mp4

```bash
for f in *.mkv; do ffmpeg -hide_banner -i "$f" -c:v libx264 -c:a copy "$f.mp4"; done
```

- `-hide_banner` → suppresses ffmpeg's long header
- `-c:v libx264` → re-encodes video in H.264 (compatible with virtually everything)
- `-c:a copy` → copies audio without re-encoding (faster and lossless)
- `"$f.mp4"` → the output name will be `video.mkv.mp4` — if you want `video.mp4`, use `"${f%.mkv}.mp4"`

Version with clean naming:

```bash
for f in *.mkv; do ffmpeg -hide_banner -i "$f" -c:v libx264 -c:a copy "${f%.mkv}.mp4"; done
```

## Performance considerations

Video conversion is CPU-intensive. If you have an NVIDIA GPU and ffmpeg compiled with NVENC support, you can speed things up significantly:

```bash
ffmpeg -i "$f" -c:v h264_nvenc -c:a copy "${f%.mkv}.mp4"
```

For Intel with Quick Sync:

```bash
ffmpeg -i "$f" -c:v h264_qsv -c:a copy "${f%.mkv}.mp4"
```

But for general use, `libx264` with `-preset fast` is already a good balance:

```bash
ffmpeg -i "$f" -c:v libx264 -preset fast -c:a copy "${f%.mkv}.mp4"
```
