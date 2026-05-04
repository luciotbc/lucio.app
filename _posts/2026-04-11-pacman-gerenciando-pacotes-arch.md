---
title: "Gerenciando pacotes no Arch com pacman e AUR"
date: 2026-04-11 14:00:00 -0300
updated: 2026-04-11 14:00:00 -0300
tags: [linux, arch, pacman, aur]
excerpt: "Referência rápida dos comandos pacman que uso no dia a dia: listar pacotes instalados, limpar cache, remover orfãos e instalar a partir de uma lista."
---

No Arch (e derivados como Manjaro), o `pacman` é o gerenciador de pacotes padrão. Junto com o `yay` para o AUR, ele cobre praticamente tudo. Aqui ficam os comandos que mais uso.

## Listar pacotes

```bash
# todos os pacotes instalados
pacman -Q

# só os instalados explicitamente (não dependências)
pacman -Qqe

# pacotes de fora do repositório oficial (inclui AUR)
pacman -Qqem
```

Os switches do `-Q` valem ser decorados:

| Flag | Significado |
|------|-------------|
| `-Q` | Consulta o banco de pacotes local |
| `-e` | Somente instalados explicitamente |
| `-t` | Exclui dependências de pacotes explícitos |
| `-n` | Exclui pacotes externos/AUR |
| `-q` | Saída resumida (só o nome) |

## Limpar cache

O pacman acumula pacotes baixados em `/var/cache/pacman/pkg/`. Vale limpar de vez em quando:

```bash
sudo pacman -Sc --noconfirm
yay -Sc --noconfirm
```

O `-Sc` remove versões antigas e mantém só a última de cada pacote. Use `-Scc` pra limpar tudo, incluindo a versão atual (mais agressivo).

## Remover pacotes orfãos

Orfãos são pacotes que foram instalados como dependência mas já não têm nenhum "dono":

```bash
sudo pacman -Rns $(pacman -Qtdq)
```

- `pacman -Qtdq` lista os orfãos
- `pacman -Rns` remove o pacote, suas dependências não usadas (`-s`) e os arquivos de configuração (`-n`)

## Instalar só o que falta de uma lista

Se você tem um arquivo com uma lista de pacotes e quer instalar sem reinstalar o que já está atualizado:

```bash
pacman -S --needed <lista de pacotes>
```

O `--needed` pula pacotes que já estão na versão mais recente, evitando reinstalações desnecessárias.
