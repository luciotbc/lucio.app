---
title: "Security scan en tu app Rails con Brakeman via Docker"
date: 2026-04-01 14:00:00 -0300
updated: 2026-04-01 14:00:00 -0300
tags: [ruby, rails, security, docker, brakeman]
excerpt: "Ejecutando Brakeman en un container, sin necesidad de instalar la gem en tu Gemfile, para un scan rápido de vulnerabilidades."
lang: es
ref: security-scan-com-brakeman-via-docker
---

[Brakeman](https://brakemanscanner.org/) es un analizador estático que detecta vulnerabilidades clásicas en Rails: SQL injection, mass-assignment, XSS, open redirect. Prefiero ejecutarlo via Docker para no contaminar el `Gemfile` con una dependencia de scan y para usar la versión más reciente sin tocar el proyecto.

```bash
docker run -v ~/work/overgrad:/code presidentbeef/brakeman --color
```

Reemplazá `~/work/overgrad` con la ruta de tu repo — monta el código en el container y ejecuta el scan.

## Para usar en CI

La misma imagen funciona como step en cualquier pipeline de CI. En GitHub Actions, por ejemplo, podés ejecutarla como container y hacer fallar el build si aparece un warning de severidad alta.

```yaml
- name: Brakeman
  run: docker run --rm -v ${{ github.workspace }}:/code presidentbeef/brakeman -w2 --no-progress
```

`-w2` filtra warnings de severidad media/alta (`-w3` solo los altos).
