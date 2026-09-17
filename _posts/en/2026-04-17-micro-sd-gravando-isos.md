---
title: "Micro SD: listing disks, writing ISOs and secure erasing"
date: 2026-04-17 12:00:00 -0300
updated: 2026-04-17 12:00:00 -0300
tags: [linux, dd, micro-sd, raspberry-pi, iso]
excerpt: "How to identify the SD card in the system, write an ISO directly via dd or xzcat, and securely erase the disk with shred."
lang: en
ref: micro-sd-gravando-isos
---

Writing a system image to an SD card or USB drive from the terminal is simpler than it looks — and faster than opening Etcher most of the time.

## Identify the correct disk

Before anything else, identifying the right device is critical. `dd` doesn't forgive mistakes:

```bash
lsblk -p
```

The `-p` flag shows the full device path (e.g., `/dev/sda`, `/dev/mmcblk0`). Look for the size matching your SD card.

## Write an ISO directly

```bash
sudo dd if=linux-2020.4-live-amd64.iso of=/dev/sda bs=4M conv=fsync status=progress && sync
```

- `if=` → input file (the ISO)
- `of=` → output device (the SD card — **without partition number**: `/dev/sda`, not `/dev/sda1`)
- `bs=4M` → transfer block size (4MB is a good balance)
- `conv=fsync` → ensures data is physically written
- `status=progress` → shows real-time progress
- `&& sync` → forces synchronization before finishing

## Write a .xz compressed ISO

Raspberry Pi images usually come compressed in `.xz`. You can decompress and write in a single command:

```bash
sudo xzcat linux-2020.4-live-amd64.iso.xz | dd of=/dev/sda bs=32M conv=fsync status=progress && sync
```

Or more generically:

```bash
xzcat ~/Downloads/<file.xz> | sudo dd of=<device> bs=32M
```

If you prefer to decompress first:

```bash
xz -d server-arm64+raspi.img.xz
```

This generates the `.img` file which can be written normally with `dd`.

## Securely erase the disk (zero fill)

To erase a disk before discarding or reusing it:

```bash
sudo shred -n 2 -z -v /dev/sda1
```

- `-n 2` → makes 2 passes with random data
- `-z` → makes a final pass with zeros (disguises the shredding)
- `-v` → verbose, shows progress

> **Warning:** for SSDs and flash storage (like SD cards), `shred` doesn't guarantee complete erasure because of wear leveling. For these cases, a single `dd if=/dev/zero` is already sufficient for practical purposes.
