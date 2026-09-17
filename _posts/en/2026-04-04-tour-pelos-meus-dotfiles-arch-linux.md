---
title: "Tour through my dotfiles: modular architecture for fast setup on Arch"
date: 2026-04-04 14:00:00 -0300
updated: 2026-04-04 14:00:00 -0300
tags: [dotfiles, arch, linux, setup, bash]
excerpt: "How I organized my dotfiles to bring up a new (or remote) Arch machine in a single command, leveraging pacman, AUR, and a module-per-folder pattern with its own install.sh."
lang: en
ref: tour-pelos-meus-dotfiles-arch-linux
---

Every time I reinstalled the system or spun up a new VM, I'd lose a few hours reapplying configs. I solved that with [a dotfiles repo](https://github.com/luciotbc/dotfiles) built around a simple idea: **one command, one ready system** — as long as the base is Arch (or Manjaro) with equivalent resources.

This post is a tour inside it: the structure, the design decisions, and why Arch is the right base for this kind of automation.

## Why Arch as the base

The choice isn't aesthetic. Arch brings three things that make automated dotfiles much more predictable:

- **Atomized packages in the official base.** Almost everything I use (`docker`, `tmux`, `neovim`, `zsh`, `firefox`) is in `pacman` directly, at the current version. No PPAs, no extra repositories, no `apt update && apt upgrade` waiting 10 minutes.
- **AUR covers the rest.** What isn't in the official repo (`visual-studio-code-bin`, `slack-desktop`, `spotify`, `postman-bin`) is in the AUR and `yay` resolves dependencies automatically. This eliminates the need for `.deb`, `.AppImage`, or `snap` packages scattered around the system.
- **Clean system out of the box.** Arch boots without a desktop, without superfluous services, without snapd or flatpakd running. What I install, I know is mine. This simplifies scripts: I can assume a minimal base state and build on top of it.

The active community closes the loop: for any AUR package, there's generally someone maintaining it, even for obscure tools. This reduces the risk of depending on a binary that'll die on the next upgrade.

## The project tree

```
dotfiles/
├── _setup.sh           # remote bootstrap: clones the repo and runs install
├── install.sh          # main orchestrator
├── helpers.sh          # shared functions (echo_*, _update, _install, _symlink)
├── packages.sh         # PKG (pacman) and AUR (yay) arrays
├── docker-compose.yaml
├── README.md
├── LICENSE.txt
│
├── albert/             # launcher
├── asdf/               # version manager (erlang, elixir, node, ruby)
├── config/             # fonts, hosts
├── docker/
├── git/
├── heroku/
├── lscripts/           # utility scripts (reposition window, reconnect headset)
├── node/
├── ruby/
├── ssh/
├── terminator/
├── tmux/
├── xfce/               # desktop env
├── yay/                # AUR helper
└── zsh/                # oh-my-zsh + p10k + plugins
```

Each folder at the root is a **module** with a single responsibility. All of them have the same signature: an `install.sh` that knows how to install itself.

## The entry point: `_setup.sh`

The entry point is a one-liner to run on a new machine:

```bash
curl -sL https://raw.githubusercontent.com/luciotbc/dotfiles/master/_setup.sh | bash
```

`_setup.sh` does three things:

```bash
DOTFILES=${DOTFILES:-~/.dotfiles}
REPO=${REPO:-luciotbc/dotfiles}
REMOTE=${REMOTE:-https://github.com/${REPO}.git}
BRANCH=${BRANCH:-master}
```

Variables with defaults and env overrides (`DOTFILES=/tmp/df bash _setup.sh` works). Then it:

1. Checks if git exists; if not, aborts with a colored error.
2. Does `git clone --depth=1` into `~/.dotfiles`, with flags to avoid messing up EOLs (`core.eol=lf`, `core.autocrlf=false`) — important because shell scripts break with CRLF.
3. Enters the cloned directory and runs `./install.sh`.

Small but useful detail: it doesn't allow reinstalling over an existing installation — if `~/.dotfiles` already exists, it aborts instead of trying to merge.

## The orchestrator: `install.sh` + `helpers.sh`

The root `install.sh` has **20 lines** and is deliberately simple:

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

The intelligence lives in `helpers.sh`. The three main functions:

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

`_update` uses `pacman-mirrors --geoip` to choose the fastest mirror for the current geolocation — in practice, this makes a huge difference when you spin up a machine in another country.

`_install` is polymorphic: pass `core` to run over `${PKG[@]}` with `pacman`, or `aur` to run over `${AUR[@]}` with `yay`. The `--needed --noconfirm` flags guarantee idempotency (already-installed packages are skipped, no prompt).

`_symlink` is the heart of the modularity: it iterates through every folder at the root and runs the `install.sh` inside it. Adding a new module is just creating a folder with an `install.sh` — the orchestrator discovers it automatically.

## The declarative list: `packages.sh`

Instead of scattering `pacman -S` calls throughout scripts, everything from the official repo becomes a Bash list:

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
  # ... ~80 packages
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

Advantages of this separation:

- **Commenting = removing from the template.** I don't have to decide between Brave and Chrome forever — I comment the line on a specific machine.
- **Clean diff between machines.** If I fork for desktop and for laptop, the diff is concentrated in `packages.sh`, without changing shell scripts.
- **Single source of truth.** Everything that enters the system via package manager is in one place.

## The module pattern

Each `<module>/install.sh` follows a minimal skeleton. Example of `git/install.sh`:

```bash
#!/bin/bash

# shellcheck source=helpers.sh
. ../helpers.sh

echo_info "Symlink ~/.gitconfig"
ln -sfT "$HOME/.dotfiles/git/gitconfig" "$HOME/.gitconfig"

echo_done "Git configuration!"
```

The strategy is always the same: the config file **lives versioned inside the repo** (`git/gitconfig`), and the install script just creates a symlink in `$HOME` pointing to it. This means:

1. A `git pull` in `~/.dotfiles` updates the config on all machines.
2. Editing `~/.gitconfig` actually edits the repo file, so local changes are traceable in git.
3. To "uninstall" a module, just delete the symlink — the original file stays in the repo.

Modules that need more than a symlink do more — but keep the same `install.sh` entry point.

### `zsh/install.sh` — plugin clone + symlink

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

Installs oh-my-zsh in batch mode (no interactive prompt), clones the three plugins/themes I use (autosuggestions, syntax-highlighting, powerlevel10k), symlinks my `.zshrc` and `.p10k.zsh`, and switches the default shell to zsh — both for the user and for root.

### `asdf/install.sh` — version manager + languages

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
# ... same for erlang, elixir, node
```

asdf clones into `~/.asdf` and checks out the latest stable tag. Language plugins become declarative installation via versioned `.tool-versions`.

### `yay/install.sh` — AUR bootstrap

```bash
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
cd ..
rm -rf yay
```

This one is special: it needs to run **before** `_install aur`, because `yay` is the helper that will install everything from the AUR afterwards. That's why it appears between `_install core` and `_install aur` in the main flow — it's the "manual" part of the bootstrap.

## The complete flow, from `curl` to a ready machine

Sequence of what happens when you run the one-liner on a fresh Arch:

1. `_setup.sh` downloads itself via curl, validates `git`, clones the repo into `~/.dotfiles`.
2. The root `install.sh` begins.
3. `_update` adjusts the mirror via `geoip` and runs `pacman -Syyu`.
4. `_install core` iterates through `${PKG[@]}` installing everything from the official repo.
5. `_symlink` iterates through the folders and fires each `install.sh` (including the one for `yay`).
6. Each module handles its own dotfile: zsh changes the shell, asdf compiles Ruby/Elixir, git links `.gitconfig`, etc.
7. Finally, `_install aur` iterates through `${AUR[@]}` via `yay`.

At the end, after login, p10k is already looking good, `ruby -v` returns 2.7.1, `docker --version` responds, and VS Code is in the menu.

## Reapplying on a remote machine

The case the project was designed for: new Arch VPS, or Manjaro on a reinstalled laptop. The minimum setup on my end is:

```bash
# 1. As root, create user and give sudo
useradd -m -G wheel -s /bin/bash lucio
passwd lucio
visudo  # uncomment %wheel

# 2. Log in as the user and run the one-liner
curl -sL https://raw.githubusercontent.com/luciotbc/dotfiles/master/_setup.sh | bash

# 3. Configure SSH/GPG (instructions in README)
ssh-keygen -t rsa -b 4096 -C "hi@lucio.app"
gpg --default-new-key-algo rsa4096 --gen-key
```

The assumption is that the machine has equivalent resources — memory to compile Erlang/Ruby via asdf, space for the package set (around 6–8 GB with everything), and decent bandwidth for the initial pull. For a leaner machine, you can comment out packages in `packages.sh` or heavy modules before running.

## What I would change today

Looking at it carefully, three things I'd improve:

- **Stronger idempotency.** Today the module `install.sh` scripts assume a clean state. Running twice can fail on `git clone` (asdf, oh-my-zsh) because the folder already exists. A check like `[ -d ~/.asdf ] || git clone ...` fixes it.
- **Pinned language versions in a separate file.** Today the versions (Ruby 2.7.1, Node 10.16.0) are hardcoded in `asdf/install.sh`. Moving them to `.tool-versions` and using just `asdf install` without arguments would be cleaner.
- **Tests in a container.** The `.dockerdev/Dockerfile` that exists in the repo provides the base for running the full setup in an Arch container and validating it finishes without errors — good for CI.

## Summary of what I think this project gets right

The thing that has served me most after almost a year using this layout:

- **Modularity by folder** eliminates the need for a dotfiles framework (Dotbot, GNU Stow, chezmoi). Each module is independent, reads like configuration code, not like abstract declarative configuration.
- **Separate `packages.sh`** makes "what is installed" auditable in a single file.
- **Symlink-first** ensures my config lives in git, not in a sporadic backup.
- **Remote bootstrap** turns a new machine into a 10-minute problem, not an afternoon one.

And most importantly: leveraging Arch + AUR as a universal software repository solves what in other distros becomes a 4-tool solution (apt + snap + flatpak + AppImage). This drastically reduces what I need to code in the dotfiles — almost everything is "add the package name to the array".

Public repo at [github.com/luciotbc/dotfiles](https://github.com/luciotbc/dotfiles) if you want to fork it.
