---
title: "GRUB: restoring dual boot with Windows and installing a theme"
date: 2026-04-14 10:30:00 -0300
updated: 2026-04-14 10:30:00 -0300
tags: [linux, grub, dual-boot, windows, manjaro]
excerpt: "How to get GRUB to recognize Windows again after an update, and how to install the CyberRe theme to give your boot screen more personality."
lang: en
ref: grub-dual-boot-e-temas
---

After updating the kernel or GRUB, it's common for Windows to disappear from the boot list. `os-prober` fixes it — but it needs some extra configuration to work.

## Restoring Windows in GRUB

### 1. Enable os-prober

GRUB disabled `os-prober` by default in recent versions. Edit the configuration:

```bash
sudo nano /etc/default/grub
```

Add or change this line:

```
GRUB_DISABLE_OS_PROBER="false"
```

### 2. Mount the Windows partition (if necessary)

`os-prober` needs to see the NTFS partition mounted. If it isn't mounted, do that first — you'll need `ntfs-3g`:

```bash
sudo ntfs-3g /dev/nvme0n1p4 /mnt/win
```

Replace `/dev/nvme0n1p4` with the correct partition. Use `lsblk` to identify which one belongs to Windows.

### 3. Run os-prober and regenerate GRUB

```bash
sudo os-prober
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

`os-prober` detects the installed operating systems. `grub-mkconfig` regenerates the boot configuration file. After that, reboot and Windows should appear.

> **Manjaro note:** on Manjaro there is the alias `sudo update-grub` which does the same thing as `grub-mkconfig`. On other distros, use the full command.

---

## Installing the CyberRe theme

[CyberRe](https://www.gnome-look.org/p/1420727/) is a cyberpunk theme for the GRUB boot screen. It looks great.

```bash
cd ~/Downloads
tar -xvf "Grub2-theme CyberRe 1.0.0.tar.gz"
cd "CyberRe 1.0.0"
sudo ./install.sh
```

The install script takes care of copying the files to the right place and updating GRUB. Reboot to see the result.
