---
title: "GRUB: recuperando o dual boot com Windows e instalando tema"
date: 2026-04-14 10:30:00 -0300
updated: 2026-04-14 10:30:00 -0300
tags: [linux, grub, dual-boot, windows, manjaro]
excerpt: "Como fazer o GRUB voltar a reconhecer o Windows depois de uma atualização, e como instalar o tema CyberRe para deixar o boot com mais personalidade."
---

Depois de atualizar o kernel ou o GRUB, é comum o Windows sumir da lista de boot. O `os-prober` resolve — mas precisa de uma configuração extra pra funcionar.

## Recuperando o Windows no GRUB

### 1. Habilitar o os-prober

O GRUB desativou o `os-prober` por padrão em versões recentes. Edite a configuração:

```bash
sudo nano /etc/default/grub
```

Adicione ou altere essa linha:

```
GRUB_DISABLE_OS_PROBER="false"
```

### 2. Montar a partição do Windows (se necessário)

O `os-prober` precisa ver a partição NTFS montada. Se não estiver montada, faça isso primeiro — você vai precisar do `ntfs-3g`:

```bash
sudo ntfs-3g /dev/nvme0n1p4 /mnt/win
```

Substitua `/dev/nvme0n1p4` pela partição correta. Use `lsblk` pra identificar qual é a do Windows.

### 3. Rodar o os-prober e regenerar o GRUB

```bash
sudo os-prober
sudo grub-mkconfig -o /boot/grub/grub.cfg
```

O `os-prober` detecta os sistemas instalados. O `grub-mkconfig` regenera o arquivo de configuração do boot. Depois disso, reinicie e o Windows deve aparecer.

> **Nota Manjaro:** no Manjaro existe o alias `sudo update-grub` que faz a mesma coisa que o `grub-mkconfig`. Em outras distros, use o comando completo.

---

## Instalando o tema CyberRe

O [CyberRe](https://www.gnome-look.org/p/1420727/) é um tema cyberpunk pra tela de boot do GRUB. Fica bem bacana.

```bash
cd ~/Downloads
tar -xvf "Grub2-theme CyberRe 1.0.0.tar.gz"
cd "CyberRe 1.0.0"
sudo ./install.sh
```

O script de instalação já cuida de copiar os arquivos para o lugar certo e atualizar o GRUB. Reinicie pra ver o resultado.
