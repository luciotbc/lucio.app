---
title: "Troubleshooting de rede: do ping que não funciona ao nmcli"
date: 2026-04-14 14:00:00 -0300
updated: 2026-04-14 14:00:00 -0300
tags: [linux, rede, networking, nmcli, wifi]
excerpt: "Sequência de diagnóstico quando a rede conecta mas o ping não funciona, mais referência completa de comandos de rede: ip, rfkill, nmcli, traceroute e mais."
---

"Conectou no Wi-Fi mas a internet não vai." Situação clássica. Aqui está minha sequência de diagnóstico — e uma referência de todos os comandos de rede que uso com frequência.

## Sequência de diagnóstico

### 1. Checar os suspeitos óbvios

- VPN com kill switch ativado? Desative e teste.
- Wi-Fi desligado pelo atalho físico do notebook? No Dell é `Fn+PrtScrn`.

### 2. Verificar o estado da interface de rede

```bash
# lista todas as interfaces
ip link

# ver o status de uma interface específica
ip link show wlp3s0
```

Se a interface estiver com status `DOWN`:

```bash
sudo ip link set wlp3s0 up
```

### 3. Checar bloqueio por rfkill

O `rfkill` controla "soft blocks" — desativações por software ou atalho de teclado:

```bash
sudo rfkill list all
```

Se aparecer `Soft blocked: yes`, o Wi-Fi foi desativado pelo atalho de teclado. `Fn+PrtScrn` no Dell ativa/desativa.

### 4. Reiniciar serviços de rede

```bash
sudo systemctl restart NetworkManager
sudo systemctl restart dhcpcd
sudo systemctl restart systemd-resolved
```

## Testes de conectividade

```bash
# verificar configuração do dnsmasq
dnsmasq --test

# ver todas as rotas de rede e gateways
ip route show

# listar todos os hosts locais (/etc/hosts)
getent hosts

# rastrear o caminho até um servidor
traceroute google.com

# info de domínio
whois lucio.app
```

## Gerenciar Wi-Fi com nmcli

O `nmcli` é o cliente de terminal do NetworkManager — mais robusto que mexer nos serviços diretamente:

```bash
# listar redes disponíveis
nmcli device wifi list

# listar dispositivos de rede
nmcli device show

# ver conexões salvas
nmcli connection

# conectar a uma rede
nmcli device wifi connect nome_da_rede
```

## Referência rápida de interfaces

No meu Dell, as interfaces têm esses nomes:

| Interface | Tipo | Módulo |
|-----------|------|--------|
| `wlp3s0` | Wi-Fi da placa interna | `ath10k_pci` |
| `wlp0s20f0u4` | Adaptador Bluetooth/USB | `rtl8192` |

Os nomes variam por máquina — `ip link` sempre revela os corretos.

## Listar drivers de Wi-Fi instalados

```bash
inxi -Nazy
iwconfig
```

O `inxi -Nazy` dá um resumo bonito dos dispositivos de rede e drivers. O `iwconfig` mostra informações de interfaces wireless no estilo antigo (ainda útil).
