---
title: "Drivers and kernel modules: diagnosis and control"
date: 2026-04-13 14:00:00 -0300
updated: 2026-04-13 14:00:00 -0300
tags: [linux, kernel, drivers, modprobe, dmesg]
excerpt: "How to list installed drivers, see what the kernel recognized, load/unload modules, and debug via dmesg."
lang: en
ref: drivers-modulos-kernel
---

When some hardware doesn't work on Linux — Wi-Fi, Bluetooth, sound card — the problem almost always lies in the driver (kernel module). Here are the commands for diagnosing and intervening.

## Listing drivers and devices

```bash
# list all installed drivers (Manjaro/mhwd)
mhwd -l -d

# list all devices recognized by the kernel with their modules
lspci -k
```

`lspci -k` is especially useful: it shows each PCI device and which module is being used.

## Loading and unloading modules

```bash
# load a module
sudo modprobe ath10k_pci
sudo modprobe rtl8192cu
sudo modprobe btusb

# remove a module (useful for reinitializing without rebooting)
sudo modprobe -r ath10k_pci
```

Removing and reloading a module is the equivalent of "turning it off and on again" for a specific driver — it fixes a lot of things without needing to reboot the machine.

## Reading the kernel log with dmesg

`dmesg` shows the boot log and kernel events, including driver errors:

```bash
# full log
sudo dmesg

# filter by firmware
sudo dmesg | grep firmware

# filter by specific module
sudo dmesg | grep ath10k_pci
sudo dmesg | grep rtl8192
```

When a module fails to load, the error shows up here. Look for words like `failed`, `error`, or `firmware` to find the problem.

## Microphone fix (PulseAudio)

If the microphone stops working after some update, a PulseAudio reset usually fixes it:

```bash
cp /etc/pulse/default.pa ~/.config/pulse/default.pa
killall pulseaudio
systemctl --user disable pulseaudio
systemctl --user enable pulseaudio
```

This copies the default configuration to the user directory, kills the current process, and restarts the service.
