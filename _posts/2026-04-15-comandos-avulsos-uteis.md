---
title: "Comandos avulsos que sempre esqueço"
date: 2026-04-15 11:00:00 -0300
updated: 2026-04-15 11:00:00 -0300
tags: [linux, ffmpeg, bash, dicas]
excerpt: "Três comandos que uso esporadicamente e sempre preciso pesquisar: ver quem está usando uma porta, cortar um vídeo com ffmpeg e identificar a distro por script."
---

Alguns comandos eu uso tão raramente que toda vez que preciso tenho que ir procurar de novo. Anotando aqui pra não perder.

## Ver quem está usando uma porta

```bash
lsof -n -i4TCP:8081
```

Substitua `8081` pela porta que você quer investigar. O `-n` evita resolução de hostname (mais rápido). A saída mostra o processo, PID e usuário que está escutando naquela porta.

## Cortar um trecho de vídeo com ffmpeg

```bash
ffmpeg -i video_original.mp4 -ss 00:58:00 -t 00:00:15 -c copy video_cortado.mp4
```

- `-ss 00:58:00` → começa em 58 minutos
- `-t 00:00:15` → duração de 15 segundos
- `-c copy` → copia os streams sem re-encodar (rápido e sem perda de qualidade)

Útil pra extrair clipes ou criar amostras de vídeos longos.

## Identificar a distro por script

Quando você está num script e precisa saber em qual distro está rodando:

```bash
awk -F= '$1=="ID" { print $2 ;}' /etc/os-release
```

Retorna algo como `arch`, `ubuntu`, `debian`, `manjaro`. Dá pra usar em condicionais:

```bash
DISTRO=$(awk -F= '$1=="ID" { print $2 ;}' /etc/os-release)

if [ "$DISTRO" = "arch" ]; then
    pacman -S --needed pacote
elif [ "$DISTRO" = "ubuntu" ]; then
    apt install -y pacote
fi
```

## Bônus: fechar a tampa sem suspender

Se você usa o notebook como servidor e não quer que ele suspenda ao fechar a tampa:

```bash
sudo nvim /etc/systemd/logind.conf
# adicionar:
# HandleLidSwitch=ignore

sudo nvim /etc/UPower/UPower.conf
# adicionar:
# IgnoreLid=true
```

Útil pra deixar o note processando algo com a tampa fechada.
