---
title: "GRUB: recuperando el dual boot con Windows e instalando un tema"
date: 2026-04-14 10:30:00 -0300
updated: 2026-04-14 10:30:00 -0300
tags: [linux, grub, dual-boot, windows, manjaro]
excerpt: "Cómo hacer que GRUB vuelva a reconocer Windows después de una actualización, y cómo instalar el tema CyberRe para darle más personalidad a la pantalla de arranque."
lang: es
ref: grub-dual-boot-e-temas
---

Después de actualizar el kernel o GRUB, es común que Windows desaparezca de la lista de arranque. `os-prober` lo resuelve — pero necesita una configuración extra para funcionar.

## Recuperando Windows en GRUB

### 1. Habilitar os-prober

GRUB desactivó `os-prober` por defecto en versiones recientes. Editá la configuración:

```bash
sudo nano /etc/default/grub
```

Agregá o modificá esta línea:

```
GRUB_DISABLE_OS_PROBER="false"
```

### 2. Montar la partición de Windows (si es necesario)

`os-prober` necesita ver la partición NTFS montada. Si no está montada, hacelo primero — necesitarás `ntfs-3g`:

```bash
sudo ntfs-3g /dev/nvme0n1p4 /mnt/win
```

Reemplazá `/dev/nvme0n1p4` por la partición correcta. Usá `lsblk` para identificar cuál es la de Windows.

### 3. Ejecutar os-prober y regenerar GRUB

```bash
sudo os-prober
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

`os-prober` detecta los sistemas operativos instalados. `grub-mkconfig` regenera el archivo de configuración del arranque. Después de eso, reiniciá y Windows debería aparecer.

> **Nota Manjaro:** en Manjaro existe el alias `sudo update-grub` que hace lo mismo que `grub-mkconfig`. En otras distros, usá el comando completo.

---

## Instalando el tema CyberRe

[CyberRe](https://www.gnome-look.org/p/1420727/) es un tema cyberpunk para la pantalla de arranque de GRUB. Queda muy bien.

```bash
cd ~/Downloads
tar -xvf "Grub2-theme CyberRe 1.0.0.tar.gz"
cd "CyberRe 1.0.0"
sudo ./install.sh
```

El script de instalación se encarga de copiar los archivos al lugar correcto y actualizar GRUB. Reiniciá para ver el resultado.
