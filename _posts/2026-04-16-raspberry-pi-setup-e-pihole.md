---
title: "Raspberry Pi: setup inicial e Pi-hole"
date: 2026-04-16 11:30:00 -0300
updated: 2026-04-16 11:30:00 -0300
published: false # não publiquei ainda porque quero fazer algo mais completo, mas já deixo aqui o rascunho
tags: [linux, raspberry-pi, pi-hole, homelab]
excerpt: "Primeiros comandos após ligar um Raspberry Pi e como instalar o Pi-hole para bloquear anúncios em toda a rede doméstica."
---

O Raspberry Pi é uma ótima plataforma pra rodar serviços caseiros. Aqui está o setup básico e a instalação do Pi-hole, que bloqueia anúncios a nível de DNS pra toda a rede.

## Configuração inicial

Depois de botar o sistema no cartão SD e ligar o Pi, o primeiro passo é o `raspi-config`:

```bash
sudo raspi-config
```

Pelo menu interativo dá pra configurar: hostname, senha do usuário, habilitar SSH, configurar locale/timezone, expandir o sistema de arquivos para ocupar todo o cartão SD, entre outras coisas.

## Atualizar o sistema

```bash
sudo apt update
sudo apt upgrade
```

Simples e necessário antes de instalar qualquer coisa.

## Instalar o Pi-hole

O Pi-hole é um servidor DNS que filtra domínios de anúncios e trackers. Toda requisição DNS da rede passa por ele, e os domínios na lista de bloqueio são descartados.

```bash
curl -sSL https://install.pi-hole.net | bash
```

O instalador é interativo e guia o processo. Ao final, você recebe o endereço do painel de administração web e a senha de acesso.

**Configuração na rede:** depois de instalado, basta apontar o DNS do seu roteador para o IP do Raspberry Pi. Todos os dispositivos da rede passarão a usar o Pi-hole automaticamente.

## Bloquear anúncios do YouTube

O Pi-hole por si só não bloqueia anúncios do YouTube (eles usam o mesmo domínio que os vídeos). Para isso existe uma lista de bloqueio específica:

[github.com/kboghdady/youTube_ads_4_pi-hole](https://github.com/kboghdady/youTube_ads_4_pi-hole)

As instruções de instalação estão no repositório — basicamente é adicionar a lista de hosts ao Pi-hole e atualizar as gravity lists.
