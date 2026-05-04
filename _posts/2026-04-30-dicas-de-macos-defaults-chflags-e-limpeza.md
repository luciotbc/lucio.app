---
title: "Dicas de macOS: defaults, chflags e limpeza de diretórios"
date: 2026-04-30 11:00:00 -0300
updated: 2026-04-30 11:00:00 -0300
tags: [macos, bash, dicas]
excerpt: "Anotações de tweaks de macOS que sempre esqueço: ajustes do sistema via defaults write, lock de arquivos com chflags, desabilitar smart quotes globalmente e uma limpeza rápida de diretórios."
---

Compilando aqui algumas anotações soltas que tenho sobre macOS — ajustes que faço sempre que configuro uma máquina nova ou que preciso aplicar de novo depois de uma atualização.

## Limpando lixo de diretórios

Quando você baixa um zip ou recebe uma pasta compartilhada, costuma vir acompanhado de uma série de arquivos auxiliares que não interessam — `.DS_Store` do Finder, `Thumbs.db` do Windows, miniaturas e índices de câmeras, etc. Pra limpar tudo de uma vez:

```bash
find . -name '.DS_Store' -type f -delete
find . -name 'Thumbs.db' -type f -delete
find . -name "*.THM" -type f -delete
find . -name "*.ini" -type f -delete
find . -name "*.LRV" -type f -delete
find . -name "*.db" -type f -delete
find . -name "*.nomedia" -type f -delete
```

Depois disso, aproveite para remover diretórios vazios — primeiro liste para conferir, depois apague:

```bash
# Listar
find . -type d -empty
# Apagar
find . -type d -empty -delete
```

E um truque que sempre uso para entender o que tem dentro de uma pasta antes de mexer nela: listar todas as extensões de arquivo distintas presentes:

```bash
find . -type f -name '*.*' | sed 's|.*\.||' | sort -u
```

Útil pra decidir o que limpar ou o que processar em lote.

## Ajustes do sistema com `defaults write`

O `defaults` é o utilitário de linha de comando que lê e escreve as preferências do macOS. Boa parte das configurações que você acharia escondidas no Preferences (ou que nem aparecem na UI) são acessíveis por aqui.

### Cmd+Tab em todos os monitores

Por padrão, o seletor de aplicativos do Cmd+Tab só aparece no monitor principal. Se você usa múltiplos displays, isso é irritante — você está focado no monitor secundário, dá Cmd+Tab e o seletor aparece no outro lado da mesa. Pra fazer ele aparecer no monitor com o cursor:

```bash
defaults write com.apple.Dock appswitcher-all-displays -bool true
killall Dock
```

O `killall Dock` reinicia o Dock para a configuração ser aplicada.

### Desabilitar smart quotes e smart dashes

O macOS substitui automaticamente `"` por `“` `”` e `--` por `—` enquanto você digita. Isso é ótimo pra escrever em português no Pages, e péssimo pra escrever código ou markdown em qualquer outro lugar. Pra desligar globalmente:

```bash
defaults write NSGlobalDomain NSAutomaticQuoteSubstitutionEnabled -bool false
defaults write NSGlobalDomain NSAutomaticDashSubstitutionEnabled -bool false
defaults write com.apple.TextEdit SmartQuotes -bool false
defaults write com.apple.TextEdit SmartDashes -bool false
```

O `NSGlobalDomain` afeta o comportamento padrão de todos os apps — mas alguns apps ignoram esse global e mantêm o próprio setting. Pra forçar em todos os apps registrados no sistema, dá pra varrer:

```bash
for d in $(defaults domains | tr -d ,); do
  osascript -e "app id \"$d\"" &>/dev/null || continue
  defaults write $d SmartQuotes -bool false
  # defaults write $d SmartDashes -bool false
  # defaults write $d SmartLinks -bool false
  # defaults write $d SmartCopyPaste -bool false
  # defaults write $d TextReplacement -bool false
  # defaults write $d CheckSpellingWhileTyping -bool false
done
```

O `osascript -e "app id \"$d\""` filtra só os domínios que correspondem a apps reais — sem isso, você sai escrevendo configuração em domínios de sistema que não tem app associado. Deixei as outras linhas comentadas porque eu mesmo só desligo as smart quotes; mas dá pra ativar conforme o gosto.

## Travando o `/etc/hosts` com `chflags`

Vários malwares e até alguns instaladores de software têm o costume de mexer no `/etc/hosts`. Se você quer travar o arquivo de modo que nem o `sudo` consiga editar sem antes destravar:

```bash
sudo chflags uchg /etc/hosts && sudo chflags schg /etc/hosts
```

- `uchg` → user immutable flag (qualquer usuário fica impedido de modificar)
- `schg` → system immutable flag (nem o root modifica sem destravar; só removível em single-user mode)

Pra destravar quando precisar editar:

```bash
sudo chflags nouchg /etc/hosts && sudo chflags noschg /etc/hosts
```

Vale para qualquer arquivo, não só o `hosts`. É uma camada de proteção extra contra modificação acidental ou maliciosa.

## Desabilitar o Music.app abrindo sozinho

Esse era o que mais me incomodava: apertar play no headset e o Music.app abria do nada, mesmo que eu estivesse ouvindo Spotify ou YouTube. Tentei várias abordagens via `defaults`, `launchctl`, scripts de Automator — nada funcionou de forma confiável em versões recentes do macOS. A Apple bloqueou cada vector que existia.

A solução que funcionou pra mim foi instalar o **BetterTouchTool** e mapear as teclas de mídia para um no-op (ou para o app que eu realmente quero controlar). Não é uma solução nativa, mas é a única que sobreviveu às atualizações do sistema.

Anotando isso aqui mais como um lembrete: não vale a pena gastar tempo procurando o `defaults write` mágico que desabilita isso. Não existe mais. Use uma ferramenta externa.
