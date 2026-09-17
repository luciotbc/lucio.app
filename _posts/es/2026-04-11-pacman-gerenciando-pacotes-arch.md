---
title: "Administrando paquetes en Arch con pacman y AUR"
date: 2026-04-11 14:00:00 -0300
updated: 2026-04-11 14:00:00 -0300
tags: [linux, arch, pacman, aur]
excerpt: "Referencia rápida de los comandos de pacman que uso a diario: listar paquetes instalados, limpiar el caché, eliminar huérfanos e instalar desde una lista."
lang: es
ref: pacman-gerenciando-pacotes-arch
---

En Arch (y derivados como Manjaro), `pacman` es el gestor de paquetes predeterminado. Junto con `yay` para el AUR, cubre prácticamente todo. Estos son los comandos que más uso.

## Listar paquetes

```bash
# todos los paquetes instalados
pacman -Q

# solo los instalados explícitamente (no dependencias)
pacman -Qqe

# paquetes fuera del repositorio oficial (incluye AUR)
pacman -Qqem
```

Vale la pena memorizar los switches de `-Q`:

| Flag | Significado |
|------|-------------|
| `-Q` | Consulta la base de datos local de paquetes |
| `-e` | Solo paquetes instalados explícitamente |
| `-t` | Excluye dependencias de paquetes explícitos |
| `-n` | Excluye paquetes externos/AUR |
| `-q` | Salida resumida (solo el nombre) |

## Limpiar el caché

pacman acumula paquetes descargados en `/var/cache/pacman/pkg/`. Vale la pena limpiarlo de vez en cuando:

```bash
sudo pacman -Sc --noconfirm
yay -Sc --noconfirm
```

`-Sc` elimina versiones antiguas y conserva solo la más reciente de cada paquete. Usá `-Scc` para limpiar todo, incluyendo la versión actual (más agresivo).

## Eliminar paquetes huérfanos

Los huérfanos son paquetes que fueron instalados como dependencias pero ya no tienen ningún "dueño":

```bash
sudo pacman -Rns $(pacman -Qtdq)
```

- `pacman -Qtdq` lista los huérfanos
- `pacman -Rns` elimina el paquete, sus dependencias no usadas (`-s`) y los archivos de configuración (`-n`)

## Instalar solo lo que falta de una lista

Si tenés un archivo con una lista de paquetes y querés instalar sin reinstalar lo que ya está actualizado:

```bash
pacman -S --needed <lista de paquetes>
```

`--needed` omite los paquetes que ya están en la versión más reciente, evitando reinstalaciones innecesarias.
