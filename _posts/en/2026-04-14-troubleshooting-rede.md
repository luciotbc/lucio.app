---
title: "Network troubleshooting: from ping that doesn't work to nmcli"
date: 2026-04-14 14:00:00 -0300
updated: 2026-04-14 14:00:00 -0300
tags: [linux, rede, networking, nmcli, wifi]
excerpt: "Diagnostic sequence for when the network connects but ping doesn't work, plus a complete reference of networking commands: ip, rfkill, nmcli, traceroute, and more."
lang: en
ref: troubleshooting-rede
---

"Connected to Wi-Fi but the internet isn't working." Classic situation. Here's my diagnostic sequence — and a reference for all the networking commands I use frequently.

## Diagnostic sequence

### 1. Check the obvious suspects

- VPN with kill switch enabled? Disable it and test.
- Wi-Fi turned off by the laptop's physical shortcut? On Dell it's `Fn+PrtScrn`.

### 2. Check the network interface status

```bash
# list all interfaces
ip link

# see the status of a specific interface
ip link show wlp3s0
```

If the interface has status `DOWN`:

```bash
sudo ip link set wlp3s0 up
```

### 3. Check for rfkill block

`rfkill` controls "soft blocks" — disabling by software or keyboard shortcut:

```bash
sudo rfkill list all
```

If `Soft blocked: yes` appears, Wi-Fi was disabled by the keyboard shortcut. `Fn+PrtScrn` on Dell toggles it.

### 4. Restart network services

```bash
sudo systemctl restart NetworkManager
sudo systemctl restart dhcpcd
sudo systemctl restart systemd-resolved
```

## Connectivity tests

```bash
# check dnsmasq configuration
dnsmasq --test

# see all network routes and gateways
ip route show

# list all local hosts (/etc/hosts)
getent hosts

# trace the path to a server
traceroute google.com

# domain info
whois lucio.app
```

## Managing Wi-Fi with nmcli

`nmcli` is the terminal client for NetworkManager — more robust than dealing with services directly:

```bash
# list available networks
nmcli device wifi list

# list network devices
nmcli device show

# see saved connections
nmcli connection

# connect to a network
nmcli device wifi connect network_name
```

## Quick interface reference

On my Dell, the interfaces have these names:

| Interface | Type | Module |
|-----------|------|--------|
| `wlp3s0` | Internal Wi-Fi card | `ath10k_pci` |
| `wlp0s20f0u4` | Bluetooth/USB adapter | `rtl8192` |

Names vary by machine — `ip link` always reveals the correct ones.

## Listing installed Wi-Fi drivers

```bash
inxi -Nazy
iwconfig
```

`inxi -Nazy` gives a nice summary of network devices and drivers. `iwconfig` shows wireless interface information in the old-style format (still useful).
