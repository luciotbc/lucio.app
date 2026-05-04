---
title: "Tour pelos meus dotfiles: arquitetura modular pra setup rápido em Arch"
date: 2026-04-04 14:00:00 -0300
updated: 2026-04-04 14:00:00 -0300
tags: [dotfiles, arch, linux, setup, bash]
excerpt: "Como organizei meus dotfiles pra subir uma máquina Arch nova (ou remota) em um comando, aproveitando pacman, AUR e um padrão de módulos por pasta com install.sh próprio."
---

Toda vez que eu reinstalava o sistema ou subia uma VM nova, perdia algumas horas reaplicando configs. Resolvi isso com [um repo de dotfiles](https://github.com/luciotbc/dotfiles) construído em torno de uma ideia simples: **um comando, um sistema pronto** — desde que a base seja Arch (ou Manjaro) com recursos equivalentes.

Esse post é um tour por dentro dele: a estrutura, as decisões de design, e por que Arch é a base certa pra esse tipo de automação.

## Por que Arch como base

A escolha não é estética. Arch traz três coisas que tornam dotfiles automatizados muito mais previsíveis:

- **Pacotes atomizados na base oficial.** Quase tudo que eu uso (`docker`, `tmux`, `neovim`, `zsh`, `firefox`) está em `pacman` direto, na versão atual. Sem PPAs, sem repositórios extras, sem `apt update && apt upgrade` esperando 10 minutos.
- **AUR cobre o resto.** O que não está no oficial (`visual-studio-code-bin`, `slack-desktop`, `spotify`, `postman-bin`) está no AUR e o `yay` resolve dependências automaticamente. Isso elimina a necessidade de `.deb`, `.AppImage` ou `snap` espalhados pelo sistema.
- **Sistema limpo de fábrica.** Arch sobe sem desktop, sem serviços supérfluos, sem snapd nem flatpakd rodando. O que eu instalo eu sei que é meu. Isso simplifica scripts: posso assumir um estado base mínimo e construir em cima.

A comunidade ativa fecha o pacote: pra qualquer pacote do AUR, em geral existe gente mantendo, mesmo de tools obscuras. O que reduz o risco de eu depender de um binário que vai morrer no próximo upgrade.

## A árvore do projeto

```
dotfiles/
├── _setup.sh           # bootstrap remoto: clona o repo e dispara install
├── install.sh          # orquestrador principal
├── helpers.sh          # funções compartilhadas (echo_*, _update, _install, _symlink)
├── packages.sh         # arrays PKG (pacman) e AUR (yay)
├── docker-compose.yaml
├── README.md
├── LICENSE.txt
│
├── albert/             # launcher
├── asdf/               # version manager (erlang, elixir, node, ruby)
├── config/             # fontes, hosts
├── docker/
├── git/
├── heroku/
├── lscripts/           # scripts utilitários (recolocar janela, reconectar headset)
├── node/
├── ruby/
├── ssh/
├── terminator/
├── tmux/
├── xfce/               # desktop env
├── yay/                # AUR helper
└── zsh/                # oh-my-zsh + p10k + plugins
```

Cada pasta na raiz é um **módulo** com responsabilidade única. Todas têm a mesma assinatura: um `install.sh` que sabe se instalar sozinho.

## A entrada: `_setup.sh`

O ponto de entrada é um one-liner pra rodar em uma máquina nova:

```bash
curl -sL https://raw.githubusercontent.com/luciotbc/dotfiles/master/_setup.sh | bash
```

O `_setup.sh` faz três coisas:

```bash
DOTFILES=${DOTFILES:-~/.dotfiles}
REPO=${REPO:-luciotbc/dotfiles}
REMOTE=${REMOTE:-https://github.com/${REPO}.git}
BRANCH=${BRANCH:-master}
```

Variáveis com default e override por env (`DOTFILES=/tmp/df bash _setup.sh` funciona). Em seguida ele:

1. Verifica se git existe; se não, aborta com erro colorido.
2. Faz `git clone --depth=1` em `~/.dotfiles`, com flags pra não bagunçar EOLs (`core.eol=lf`, `core.autocrlf=false`) — importante porque scripts shell quebram com CRLF.
3. Entra no diretório clonado e roda `./install.sh`.

Detalhe pequeno mas útil: ele não permite reinstalar por cima — se `~/.dotfiles` já existe, ele aborta em vez de tentar mesclar.

## O orquestrador: `install.sh` + `helpers.sh`

O `install.sh` da raiz tem **20 linhas** e é deliberadamente burro:

```bash
#!/bin/bash

. packages.sh
. helpers.sh

echo_info "Updating packages..."
_update

echo_info "Installing core packages..."
_install core

echo_info "Configure settings..."
_symlink

echo_info "Installing aur packages..."
_install aur
```

A inteligência fica em `helpers.sh`. As três funções principais:

```bash
function _update() {
  sudo pacman-mirrors --geoip
  sudo pacman -Syyu --needed --noconfirm
}

function _install() {
  if [[ $1 == "core" ]]; then
    for pkg in "${PKG[@]}"; do
      sudo pacman -Sy "$pkg" --needed --noconfirm
      echo_done "${pkg} installed!"
    done
  elif [[ $1 == "aur" ]]; then
    for aur in "${AUR[@]}"; do
      yay -S "$aur" --needed --noconfirm
    done
  fi
}

function _symlink() {
  dirs=$(find . -maxdepth 1 -mindepth 1 -type d -not -name '.git')

  for dir in $dirs; do
    cd "$dir" || exit
    ./install.sh
    cd ..
  done
}
```

`_update` usa `pacman-mirrors --geoip` pra escolher o mirror mais rápido pra geolocalização atual — na prática, faz uma diferença gigante quando você sobe uma máquina em outro país.

`_install` é polimórfica: passa `core` pra rodar sobre `${PKG[@]}` com `pacman`, ou `aur` pra rodar sobre `${AUR[@]}` com `yay`. As flags `--needed --noconfirm` garantem idempotência (pacote já instalado é ignorado, sem prompt).

`_symlink` é o coração da modularidade: itera por toda pasta na raiz e roda o `install.sh` que estiver lá. Adicionar um módulo novo é só criar uma pasta com `install.sh` — o orquestrador descobre sozinho.

## A lista declarativa: `packages.sh`

Em vez de espalhar `pacman -S` pelos scripts, tudo que vem do oficial vira uma lista Bash:

```bash
export PKG=(
  albert
  base-devel
  blueman
  chromium
  cmake
  docker
  docker-compose
  firefox
  fzf
  htop
  neovim
  noto-fonts
  openssh
  postgresql-libs
  terminator
  tmux
  zsh
  # ... ~80 pacotes
)

export AUR=(
  brave-bin
  ctop
  espanso
  git-delta
  google-chrome
  insync
  postman-bin
  slack-desktop
  spotify
  visual-studio-code-bin
  zeal
)
```

Vantagens dessa separação:

- **Comentar = desinstalar do template.** Não preciso decidir entre Brave e Chrome pra sempre — comento a linha numa máquina específica.
- **Diff limpa entre máquinas.** Se eu fizer fork pra desktop e pra notebook, o diff fica concentrado em `packages.sh`, sem mudar shell scripts.
- **Source única da verdade.** Tudo que entra no sistema via gerenciador de pacote está num lugar só.

## O padrão dos módulos

Cada `<modulo>/install.sh` segue um esqueleto mínimo. Exemplo do `git/install.sh`:

```bash
#!/bin/bash

# shellcheck source=helpers.sh
. ../helpers.sh

echo_info "Symlink ~/.gitconfig"
ln -sfT "$HOME/.dotfiles/git/gitconfig" "$HOME/.gitconfig"

echo_done "Git configuration!"
```

A estratégia é sempre a mesma: o arquivo de config **mora versionado dentro do repo** (`git/gitconfig`), e o install só cria um symlink em `$HOME` apontando pra ele. Isso quer dizer que:

1. Um `git pull` no `~/.dotfiles` atualiza a config em todas as máquinas.
2. Editar `~/.gitconfig` na verdade edita o arquivo do repo, então mudanças locais ficam rastreáveis no git.
3. Pra "desinstalar" um módulo, basta apagar o symlink — o arquivo original continua no repo.

Os módulos que precisam de mais coisa que symlink fazem mais — mas mantêm a mesma porta de entrada `install.sh`.

### `zsh/install.sh` — clone de plugins + symlink

```bash
sh -c "$(curl -fsSL .../oh-my-zsh/.../install.sh)" -s --batch
git clone --depth=1 https://github.com/zsh-users/zsh-autosuggestions \
  "$HOME/.oh-my-zsh/custom/plugins/zsh-autosuggestions"
git clone --depth=1 https://github.com/zsh-users/zsh-syntax-highlighting.git \
  "$HOME/.oh-my-zsh/custom/plugins/zsh-syntax-highlighting"
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git \
  "$HOME/.oh-my-zsh/themes/powerlevel10k"

ln -sfT "$HOME/.dotfiles/zsh/.p10k.zsh" "$HOME/.p10k.zsh"
ln -sfT "$HOME/.dotfiles/zsh/zshrc" "$HOME/.zshrc"

chsh -s $(which zsh)
sudo chsh -s $(which zsh)
```

Instala oh-my-zsh em modo batch (sem prompt interativo), clona os três plugins/themes que eu uso (autosuggestions, syntax-highlighting, powerlevel10k), faz symlink dos meus `.zshrc` e `.p10k.zsh`, e troca o shell padrão pra zsh — tanto pro usuário quanto pro root.

### `asdf/install.sh` — version manager + linguagens

```bash
git clone https://github.com/asdf-vm/asdf.git "$HOME/.asdf"
cd "$HOME/.asdf" && git checkout "$(git describe --abbrev=0 --tags)"

ln -sfT "$HOME/.dotfiles/asdf/asdfrc" "$HOME/.asdfrc"
ln -sfT "$HOME/.dotfiles/asdf/tool-versions" "$HOME/.tool-versions"

~/.asdf/bin/asdf plugin-add erlang
~/.asdf/bin/asdf plugin-add elixir
~/.asdf/bin/asdf plugin-add nodejs
~/.asdf/bin/asdf plugin-add ruby

~/.asdf/bin/asdf install ruby 2.7.1
~/.asdf/bin/asdf global ruby 2.7.1
# ... mesma coisa pra erlang, elixir, node
```

asdf clona em `~/.asdf` e checa out na última tag estável. Os plugins de linguagem viram instalação declarativa via `.tool-versions` versionado.

### `yay/install.sh` — bootstrap do AUR

```bash
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
cd ..
rm -rf yay
```

Esse é especial: precisa rodar **antes** do `_install aur`, porque `yay` é o helper que vai instalar tudo do AUR depois. Por isso ele aparece entre o `_install core` e o `_install aur` no fluxo principal — é a parte "manual" do bootstrap.

## O fluxo completo, do `curl` até a máquina pronta

Sequência do que acontece quando você roda o one-liner numa Arch zerada:

1. `_setup.sh` baixa-se via curl, valida `git`, clona o repo em `~/.dotfiles`.
2. `install.sh` da raiz começa.
3. `_update` ajusta mirror via `geoip` e roda `pacman -Syyu`.
4. `_install core` percorre `${PKG[@]}` instalando tudo do oficial.
5. `_symlink` itera pelas pastas e dispara cada `install.sh` (incluindo o do `yay`).
6. Cada módulo cuida do próprio dotfile: zsh muda shell, asdf compila Ruby/Elixir, git linka `.gitconfig`, etc.
7. Por último, `_install aur` percorre `${AUR[@]}` via `yay`.

No fim, login, p10k já está bonito, `ruby -v` retorna 2.7.1, `docker --version` responde, e o VS Code está no menu.

## Reaplicando em uma máquina remota

O caso pra qual o projeto foi pensado: VPS Arch nova, ou Manjaro num notebook reinstalado. O setup mínimo do meu lado é:

```bash
# 1. Como root, criar usuário e dar sudo
useradd -m -G wheel -s /bin/bash lucio
passwd lucio
visudo  # descomentar %wheel

# 2. Logar como o usuário e rodar o one-liner
curl -sL https://raw.githubusercontent.com/luciotbc/dotfiles/master/_setup.sh | bash

# 3. Configurar SSH/GPG (instruções no README)
ssh-keygen -t rsa -b 4096 -C "hi@lucio.app"
gpg --default-new-key-algo rsa4096 --gen-key
```

A premissa é que a máquina tem recursos equivalentes — memória pra compilar Erlang/Ruby via asdf, espaço pra o conjunto de pacotes (uns 6–8 GB com tudo) e rede decente pra o initial pull. Se for uma máquina mais magra, dá pra comentar pacotes em `packages.sh` ou módulos pesados antes de rodar.

## O que eu mudaria hoje

Olhando com calma, três coisas que melhoraria:

- **Idempotência mais forte.** Hoje `install.sh` dos módulos assume estado limpo. Rodar duas vezes pode falhar nos `git clone` (asdf, oh-my-zsh) porque a pasta já existe. Um check do tipo `[ -d ~/.asdf ] || git clone ...` resolve.
- **Versões pinadas das linguagens em arquivo separado.** Hoje as versões (Ruby 2.7.1, Node 10.16.0) estão hardcoded em `asdf/install.sh`. Mover pra `.tool-versions` e usar só `asdf install` sem argumentos seria mais limpo.
- **Testes em container.** O `.dockerdev/Dockerfile` que existe no repo dá a base pra rodar o setup completo num container Arch e validar que ele termina sem erro — bom pra CI.

## Resumo do que eu acho que esse projeto acerta

A coisa que mais me serve depois de quase um ano usando esse layout:

- **Modularidade por pasta** elimina a necessidade de um framework de dotfiles (Dotbot, GNU Stow, chezmoi). Cada módulo é independente, lê como código de configuração, não como configuração declarativa abstrata.
- **`packages.sh` separado** torna o "o que está instalado" auditável num arquivo só.
- **Symlink-first** garante que minha config vive no git, não num backup esporádico.
- **Bootstrap remoto** torna a máquina nova um problema de 10 minutos, não de uma tarde.

E principalmente: aproveitar Arch + AUR como repositório universal de software resolve o que em outras distros vira solução com 4 ferramentas diferentes (apt + snap + flatpak + AppImage). Isso reduz drasticamente o que eu preciso codar nos dotfiles — quase tudo é "adicionar nome do pacote no array".

Repo público em [github.com/luciotbc/dotfiles](https://github.com/luciotbc/dotfiles) se quiser fazer fork.
