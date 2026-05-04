---
title: "Micro SD: listando discos, gravando ISOs e apagando com segurança"
date: 2026-04-17 12:00:00 -0300
updated: 2026-04-17 12:00:00 -0300
tags: [linux, dd, micro-sd, raspberry-pi, iso]
excerpt: "Como identificar o cartão SD no sistema, gravar uma ISO diretamente via dd ou xzcat, e apagar o disco de forma segura com shred."
---

Gravar uma imagem de sistema num cartão SD ou pendrive pelo terminal é mais simples do que parece — e mais rápido do que abrir o Etcher na maioria das vezes.

## Identificar o disco correto

Antes de qualquer coisa, identificar o dispositivo certo é crítico. O `dd` não perdoa enganos:

```bash
lsblk -p
```

O `-p` mostra o caminho completo do dispositivo (ex: `/dev/sda`, `/dev/mmcblk0`). Procure pelo tamanho correspondente ao seu cartão SD.

## Gravar uma ISO diretamente

```bash
sudo dd if=linux-2020.4-live-amd64.iso of=/dev/sda bs=4M conv=fsync status=progress && sync
```

- `if=` → arquivo de entrada (a ISO)
- `of=` → dispositivo de saída (o SD card — **sem número de partição**: `/dev/sda`, não `/dev/sda1`)
- `bs=4M` → tamanho do bloco de transferência (4MB é um bom equilíbrio)
- `conv=fsync` → garante que os dados sejam escritos fisicamente
- `status=progress` → mostra o progresso em tempo real
- `&& sync` → força a sincronização antes de encerrar

## Gravar uma ISO comprimida em .xz

Imagens do Raspberry Pi geralmente vêm comprimidas em `.xz`. Dá pra descompactar e gravar em um único comando:

```bash
sudo xzcat linux-2020.4-live-amd64.iso.xz | dd of=/dev/sda bs=32M conv=fsync status=progress && sync
```

Ou de forma mais genérica:

```bash
xzcat ~/Downloads/<arquivo.xz> | sudo dd of=<dispositivo> bs=32M
```

Se preferir descompactar primeiro:

```bash
xz -d server-arm64+raspi.img.xz
```

Isso gera o arquivo `.img` que pode ser gravado normalmente com o `dd`.

## Apagar o disco com segurança (zero fill)

Para apagar um disco antes de descartar ou reutilizar:

```bash
sudo shred -n 2 -z -v /dev/sda1
```

- `-n 2` → faz 2 passadas com dados aleatórios
- `-z` → faz uma passada final com zeros (disfarça o shredding)
- `-v` → verbose, mostra o progresso

> **Atenção:** para SSDs e flash storage (como cartões SD), o `shred` não garante apagamento completo por causa do wear leveling. Para esses casos, um único `dd if=/dev/zero` já é suficiente para uso prático.
