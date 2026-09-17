---
title: "Managing packages on Arch with pacman and AUR"
date: 2026-04-11 14:00:00 -0300
updated: 2026-04-11 14:00:00 -0300
tags: [linux, arch, pacman, aur]
excerpt: "Quick reference of the pacman commands I use daily: listing installed packages, clearing cache, removing orphans, and installing from a list."
lang: en
ref: pacman-gerenciando-pacotes-arch
---

On Arch (and derivatives like Manjaro), `pacman` is the default package manager. Together with `yay` for the AUR, it covers pretty much everything. Here are the commands I use most.

## Listing packages

```bash
# all installed packages
pacman -Q

# only explicitly installed ones (not dependencies)
pacman -Qqe

# packages from outside the official repository (includes AUR)
pacman -Qqem
```

The `-Q` switches are worth memorizing:

| Flag | Meaning |
|------|---------|
| `-Q` | Queries the local package database |
| `-e` | Only explicitly installed packages |
| `-t` | Excludes dependencies of explicit packages |
| `-n` | Excludes external/AUR packages |
| `-q` | Quiet output (name only) |

## Clearing cache

pacman accumulates downloaded packages in `/var/cache/pacman/pkg/`. It's worth cleaning it out occasionally:

```bash
sudo pacman -Sc --noconfirm
yay -Sc --noconfirm
```

`-Sc` removes old versions and keeps only the latest of each package. Use `-Scc` to clear everything, including the current version (more aggressive).

## Removing orphan packages

Orphans are packages that were installed as dependencies but no longer have any "owner":

```bash
sudo pacman -Rns $(pacman -Qtdq)
```

- `pacman -Qtdq` lists the orphans
- `pacman -Rns` removes the package, its unused dependencies (`-s`), and configuration files (`-n`)

## Install only what's missing from a list

If you have a file with a list of packages and want to install without reinstalling what's already up to date:

```bash
pacman -S --needed <package list>
```

`--needed` skips packages that are already at the latest version, avoiding unnecessary reinstalls.
