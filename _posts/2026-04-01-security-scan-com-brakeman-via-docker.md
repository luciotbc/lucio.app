---
title: "Security scan no seu Rails com Brakeman via Docker"
date: 2026-04-01 14:00:00 -0300
updated: 2026-04-01 14:00:00 -0300
tags: [ruby, rails, security, docker, brakeman]
excerpt: "Rodando Brakeman num container, sem precisar instalar a gem no seu Gemfile, pra um scan rápido de vulnerabilidades."
---

[Brakeman](https://brakemanscanner.org/) é um analisador estático que pega buracos clássicos em Rails: SQL injection, mass-assignment, XSS, redirect aberto. Eu prefiro rodar via Docker pra não poluir o `Gemfile` com dependência de scan e pra usar a versão mais nova sem mexer no projeto.

```bash
docker run -v ~/work/overgrad:/code presidentbeef/brakeman --color
```

Trocando `~/work/overgrad` pelo caminho do seu repo, ele monta o código no container e roda o scan.

## Para colocar em CI

A mesma imagem funciona como step em qualquer CI. Em GitHub Actions, por exemplo, dá pra rodar como container e falhar o build se aparecer warning de severidade alta.

```yaml
- name: Brakeman
  run: docker run --rm -v ${{ github.workspace }}:/code presidentbeef/brakeman -w2 --no-progress
```

`-w2` filtra avisos de severidade média/alta (`-w3` só os altos).
