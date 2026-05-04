---
title: "Drivers e módulos do kernel: diagnóstico e controle"
date: 2026-04-13 14:00:00 -0300
updated: 2026-04-13 14:00:00 -0300
tags: [linux, kernel, drivers, modprobe, dmesg]
excerpt: "Como listar drivers instalados, ver o que o kernel reconheceu, carregar/descarregar módulos e debugar via dmesg."
---

Quando algum hardware não funciona no Linux — Wi-Fi, Bluetooth, placa de som — o problema quase sempre está no driver (módulo do kernel). Aqui estão os comandos pra diagnosticar e intervir.

## Listar drivers e dispositivos

```bash
# lista todos os drivers instalados (Manjaro/mhwd)
mhwd -l -d

# lista todos os dispositivos reconhecidos pelo kernel com seus módulos
lspci -k
```

O `lspci -k` é especialmente útil: mostra cada dispositivo PCI e qual módulo está sendo usado.

## Carregar e descarregar módulos

```bash
# carrega um módulo
sudo modprobe ath10k_pci
sudo modprobe rtl8192cu
sudo modprobe btusb

# remove um módulo (útil pra reinicializar sem reiniciar o sistema)
sudo modprobe -r ath10k_pci
```

Remover e recarregar um módulo é o equivalente a "desligar e ligar de novo" para um driver específico — resolve muita coisa sem precisar reiniciar a máquina.

## Ler o log do kernel com dmesg

O `dmesg` mostra o log de boot e eventos do kernel, incluindo erros de driver:

```bash
# log completo
sudo dmesg

# filtrar por firmware
sudo dmesg | grep firmware

# filtrar por módulo específico
sudo dmesg | grep ath10k_pci
sudo dmesg | grep rtl8192
```

Quando um módulo falha ao carregar, o erro aparece aqui. Procure por palavras como `failed`, `error` ou `firmware` para achar o problema.

## Fix de microfone (PulseAudio)

Se o microfone parar de funcionar depois de alguma atualização, um reset no PulseAudio costuma resolver:

```bash
cp /etc/pulse/default.pa ~/.config/pulse/default.pa
killall pulseaudio
systemctl --user disable pulseaudio
systemctl --user enable pulseaudio
```

Copia a configuração padrão para o diretório do usuário, mata o processo atual e reinicia o serviço.
