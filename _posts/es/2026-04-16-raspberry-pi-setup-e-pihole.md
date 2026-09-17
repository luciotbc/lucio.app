---
title: "Raspberry Pi: configuración inicial y Pi-hole"
date: 2026-04-16 11:30:00 -0300
updated: 2026-04-16 11:30:00 -0300
published: false # não publiquei ainda porque quero fazer algo mais completo, mas já deixo aqui o rascunho
tags: [linux, raspberry-pi, pi-hole, homelab]
excerpt: "Primeros comandos al encender una Raspberry Pi y cómo instalar Pi-hole para bloquear anuncios en toda la red doméstica."
lang: es
ref: raspberry-pi-setup-e-pihole
---

La Raspberry Pi es una excelente plataforma para correr servicios caseros. Aquí está la configuración básica y la instalación de Pi-hole, que bloquea anuncios a nivel de DNS para toda la red.

## Configuración inicial

Después de poner el sistema en la tarjeta SD y encender la Pi, el primer paso es `raspi-config`:

```bash
sudo raspi-config
```

A través del menú interactivo se puede configurar: hostname, contraseña del usuario, habilitar SSH, configurar locale/timezone, expandir el sistema de archivos para ocupar toda la tarjeta SD, entre otras cosas.

## Actualizar el sistema

```bash
sudo apt update
sudo apt upgrade
```

Simple y necesario antes de instalar cualquier cosa.

## Instalar Pi-hole

Pi-hole es un servidor DNS que filtra dominios de anuncios y trackers. Cada solicitud DNS de la red pasa por él, y los dominios en la lista de bloqueo son descartados.

```bash
curl -sSL https://install.pi-hole.net | bash
```

El instalador es interactivo y guía el proceso. Al final, recibes la dirección del panel de administración web y la contraseña de acceso.

**Configuración en la red:** una vez instalado, basta con apuntar el DNS de tu router hacia la IP de la Raspberry Pi. Todos los dispositivos de la red comenzarán a usar Pi-hole automáticamente.

## Bloquear anuncios de YouTube

Pi-hole por sí solo no bloquea los anuncios de YouTube (usan el mismo dominio que los videos). Para eso existe una lista de bloqueo específica:

[github.com/kboghdady/youTube_ads_4_pi-hole](https://github.com/kboghdady/youTube_ads_4_pi-hole)

Las instrucciones de instalación están en el repositorio — básicamente es agregar la lista de hosts a Pi-hole y actualizar las gravity lists.
