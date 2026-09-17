---
title: "Drivers y módulos del kernel: diagnóstico y control"
date: 2026-04-13 14:00:00 -0300
updated: 2026-04-13 14:00:00 -0300
tags: [linux, kernel, drivers, modprobe, dmesg]
excerpt: "Cómo listar drivers instalados, ver qué reconoció el kernel, cargar/descargar módulos y depurar con dmesg."
lang: es
ref: drivers-modulos-kernel
---

Cuando algún hardware no funciona en Linux — Wi-Fi, Bluetooth, placa de sonido — el problema casi siempre está en el driver (módulo del kernel). Estos son los comandos para diagnosticar e intervenir.

## Listar drivers y dispositivos

```bash
# lista todos los drivers instalados (Manjaro/mhwd)
mhwd -l -d

# lista todos los dispositivos reconocidos por el kernel con sus módulos
lspci -k
```

`lspci -k` es especialmente útil: muestra cada dispositivo PCI y qué módulo está siendo usado.

## Cargar y descargar módulos

```bash
# carga un módulo
sudo modprobe ath10k_pci
sudo modprobe rtl8192cu
sudo modprobe btusb

# elimina un módulo (útil para reinicializar sin reiniciar el sistema)
sudo modprobe -r ath10k_pci
```

Eliminar y recargar un módulo es el equivalente a "apagar y volver a encender" para un driver específico — resuelve muchas cosas sin necesidad de reiniciar la máquina.

## Leer el log del kernel con dmesg

`dmesg` muestra el log de arranque y los eventos del kernel, incluyendo errores de driver:

```bash
# log completo
sudo dmesg

# filtrar por firmware
sudo dmesg | grep firmware

# filtrar por módulo específico
sudo dmesg | grep ath10k_pci
sudo dmesg | grep rtl8192
```

Cuando un módulo falla al cargar, el error aparece aquí. Buscá palabras como `failed`, `error` o `firmware` para encontrar el problema.

## Fix de micrófono (PulseAudio)

Si el micrófono deja de funcionar después de alguna actualización, un reset de PulseAudio suele resolver el problema:

```bash
cp /etc/pulse/default.pa ~/.config/pulse/default.pa
killall pulseaudio
systemctl --user disable pulseaudio
systemctl --user enable pulseaudio
```

Copia la configuración predeterminada al directorio del usuario, mata el proceso actual y reinicia el servicio.
