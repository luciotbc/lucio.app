---
title: "Troubleshooting de red: del ping que no funciona al nmcli"
date: 2026-04-14 14:00:00 -0300
updated: 2026-04-14 14:00:00 -0300
tags: [linux, rede, networking, nmcli, wifi]
excerpt: "Secuencia de diagnóstico cuando la red conecta pero el ping no funciona, más referencia completa de comandos de red: ip, rfkill, nmcli, traceroute y más."
lang: es
ref: troubleshooting-rede
---

"Conectado al Wi-Fi pero internet no anda." Situación clásica. Esta es mi secuencia de diagnóstico — y una referencia de todos los comandos de red que uso con frecuencia.

## Secuencia de diagnóstico

### 1. Revisar los sospechosos obvios

- ¿VPN con kill switch activado? Desactivalo y probá.
- ¿Wi-Fi apagado por el atajo físico de la notebook? En Dell es `Fn+PrtScrn`.

### 2. Verificar el estado de la interfaz de red

```bash
# lista todas las interfaces
ip link

# ver el estado de una interfaz específica
ip link show wlp3s0
```

Si la interfaz tiene estado `DOWN`:

```bash
sudo ip link set wlp3s0 up
```

### 3. Revisar bloqueo por rfkill

`rfkill` controla los "soft blocks" — desactivaciones por software o atajo de teclado:

```bash
sudo rfkill list all
```

Si aparece `Soft blocked: yes`, el Wi-Fi fue desactivado por el atajo de teclado. `Fn+PrtScrn` en Dell activa/desactiva.

### 4. Reiniciar servicios de red

```bash
sudo systemctl restart NetworkManager
sudo systemctl restart dhcpcd
sudo systemctl restart systemd-resolved
```

## Pruebas de conectividad

```bash
# verificar configuración de dnsmasq
dnsmasq --test

# ver todas las rutas de red y gateways
ip route show

# listar todos los hosts locales (/etc/hosts)
getent hosts

# rastrear el camino hasta un servidor
traceroute google.com

# info de dominio
whois lucio.app
```

## Administrar Wi-Fi con nmcli

`nmcli` es el cliente de terminal de NetworkManager — más robusto que trabajar directamente con los servicios:

```bash
# listar redes disponibles
nmcli device wifi list

# listar dispositivos de red
nmcli device show

# ver conexiones guardadas
nmcli connection

# conectar a una red
nmcli device wifi connect nombre_de_red
```

## Referencia rápida de interfaces

En mi Dell, las interfaces tienen estos nombres:

| Interfaz | Tipo | Módulo |
|----------|------|--------|
| `wlp3s0` | Wi-Fi de la placa interna | `ath10k_pci` |
| `wlp0s20f0u4` | Adaptador Bluetooth/USB | `rtl8192` |

Los nombres varían según la máquina — `ip link` siempre revela los correctos.

## Listar drivers de Wi-Fi instalados

```bash
inxi -Nazy
iwconfig
```

`inxi -Nazy` da un resumen claro de los dispositivos de red y drivers. `iwconfig` muestra información de interfaces wireless en el estilo antiguo (sigue siendo útil).
