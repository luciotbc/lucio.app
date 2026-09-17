---
title: "Raspberry Pi: initial setup and Pi-hole"
date: 2026-04-16 11:30:00 -0300
updated: 2026-04-16 11:30:00 -0300
published: false # não publiquei ainda porque quero fazer algo mais completo, mas já deixo aqui o rascunho
tags: [linux, raspberry-pi, pi-hole, homelab]
excerpt: "First commands after booting a Raspberry Pi and how to install Pi-hole to block ads across your entire home network."
lang: en
ref: raspberry-pi-setup-e-pihole
---

The Raspberry Pi is a great platform for running home services. Here's the basic setup and the Pi-hole installation, which blocks ads at the DNS level for the whole network.

## Initial configuration

After putting the system on the SD card and powering up the Pi, the first step is `raspi-config`:

```bash
sudo raspi-config
```

Through the interactive menu you can configure: hostname, user password, enable SSH, set locale/timezone, expand the filesystem to use the full SD card, and more.

## Update the system

```bash
sudo apt update
sudo apt upgrade
```

Simple and necessary before installing anything.

## Install Pi-hole

Pi-hole is a DNS server that filters ad and tracker domains. Every DNS request on the network goes through it, and domains on the blocklist are dropped.

```bash
curl -sSL https://install.pi-hole.net | bash
```

The installer is interactive and guides you through the process. At the end, you get the address of the web admin panel and the access password.

**Network configuration:** once installed, just point your router's DNS to the Raspberry Pi's IP address. All devices on the network will start using Pi-hole automatically.

## Block YouTube ads

Pi-hole alone doesn't block YouTube ads (they use the same domain as the videos). For that there's a specific blocklist:

[github.com/kboghdady/youTube_ads_4_pi-hole](https://github.com/kboghdady/youTube_ads_4_pi-hole)

Installation instructions are in the repository — basically add the hosts list to Pi-hole and update the gravity lists.
