---
title: "Tour por mis dotfiles: arquitectura modular para setup rápido en Arch"
date: 2026-04-04 14:00:00 -0300
updated: 2026-04-04 14:00:00 -0300
tags: [dotfiles, arch, linux, setup, bash]
excerpt: "Cómo organicé mis dotfiles para levantar una máquina Arch nueva (o remota) en un comando, aprovechando pacman, AUR y un patrón de módulos por carpeta con su propio install.sh."
lang: es
ref: tour-pelos-meus-dotfiles-arch-linux
---

Cada vez que reinstalaba el sistema o levantaba una VM nueva, perdía algunas horas reaplicando configs. Lo resolví con [un repo de dotfiles](https://github.com/luciotbc/dotfiles) construido alrededor de una idea simple: **un comando, un sistema listo** — siempre que la base sea Arch (o Manjaro) con recursos equivalentes.

Este post es un tour por dentro: la estructura, las decisiones de diseño, y por qué Arch es la base correcta para este tipo de automatización.

## Por qué Arch como base

La elección no es estética. Arch trae tres cosas que hacen que los dotfiles automatizados sean mucho más predecibles:

- **Paquetes atomizados en la base oficial.** Casi todo lo que uso (`docker`, `tmux`, `neovim`, `zsh`, `firefox`) está en `pacman` directamente, en la versión actual. Sin PPAs, sin repositorios extras, sin `apt update && apt upgrade` esperando 10 minutos.
- **AUR cubre el resto.** Lo que no está en el oficial (`visual-studio-code-bin`, `slack-desktop`, `spotify`, `postman-bin`) está en el AUR y `yay` resuelve dependencias automáticamente. Esto elimina la necesidad de `.deb`, `.AppImage` o `snap` esparcidos por el sistema.
- **Sistema limpio de fábrica.** Arch arranca sin desktop, sin servicios superfluos, sin snapd ni flatpakd corriendo. Lo que instalo sé que es mío. Esto simplifica los scripts: puedo asumir un estado base mínimo y construir encima.

La comunidad activa cierra el círculo: para cualquier paquete del AUR, generalmente hay gente manteniéndolo, incluso para tools oscuras. Lo que reduce el riesgo de depender de un binario que va a morir en el próximo upgrade.

## El árbol del proyecto

```
dotfiles/
├── _setup.sh           # bootstrap remoto: clona el repo y dispara install
├── install.sh          # orquestador principal
├── helpers.sh          # funciones compartidas (echo_*, _update, _install, _symlink)
├── packages.sh         # arrays PKG (pacman) y AUR (yay)
├── docker-compose.yaml
├── README.md
├── LICENSE.txt
│
├── albert/             # launcher
├── asdf/               # version manager (erlang, elixir, node, ruby)
├── config/             # fuentes, hosts
├── docker/
├── git/
├── heroku/
├── lscripts/           # scripts utilitarios (reposicionar ventana, reconectar headset)
├── node/
├── ruby/
├── ssh/
├── terminator/
├── tmux/
├── xfce/               # desktop env
├── yay/                # AUR helper
└── zsh/                # oh-my-zsh + p10k + plugins
```

Cada carpeta en la raíz es un **módulo** con responsabilidad única. Todas tienen la misma firma: un `install.sh` que sabe instalarse solo.

## El punto de entrada: `_setup.sh`

El punto de entrada es un one-liner para correr en una máquina nueva:

```bash
curl -sL https://raw.githubusercontent.com/luciotbc/dotfiles/master/_setup.sh | bash
```

`_setup.sh` hace tres cosas:

```bash
DOTFILES=${DOTFILES:-~/.dotfiles}
REPO=${REPO:-luciotbc/dotfiles}
REMOTE=${REMOTE:-https://github.com/${REPO}.git}
BRANCH=${BRANCH:-master}
```

Variables con defaults y override por env (`DOTFILES=/tmp/df bash _setup.sh` funciona). Luego:

1. Verifica si git existe; si no, aborta con error con color.
2. Hace `git clone --depth=1` en `~/.dotfiles`, con flags para no mezclar EOLs (`core.eol=lf`, `core.autocrlf=false`) — importante porque los scripts shell se rompen con CRLF.
3. Entra al directorio clonado y corre `./install.sh`.

Detalle pequeño pero útil: no permite reinstalar sobre una instalación existente — si `~/.dotfiles` ya existe, aborta en vez de intentar mergear.

## El orquestador: `install.sh` + `helpers.sh`

El `install.sh` de la raíz tiene **20 líneas** y es deliberadamente simple:

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

La inteligencia está en `helpers.sh`. Las tres funciones principales:

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

`_update` usa `pacman-mirrors --geoip` para elegir el mirror más rápido para la geolocalización actual — en la práctica, hace una diferencia enorme cuando levantás una máquina en otro país.

`_install` es polimórfica: pasá `core` para correr sobre `${PKG[@]}` con `pacman`, o `aur` para correr sobre `${AUR[@]}` con `yay`. Los flags `--needed --noconfirm` garantizan idempotencia (paquete ya instalado se ignora, sin prompt).

`_symlink` es el corazón de la modularidad: itera por cada carpeta en la raíz y corre el `install.sh` que esté ahí. Agregar un módulo nuevo es solo crear una carpeta con `install.sh` — el orquestador lo descubre solo.

## La lista declarativa: `packages.sh`

En vez de esparcir `pacman -S` por los scripts, todo lo que viene del oficial se convierte en una lista Bash:

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
  # ... ~80 paquetes
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

Ventajas de esta separación:

- **Comentar = desinstalar del template.** No necesito decidir entre Brave y Chrome para siempre — comento la línea en una máquina específica.
- **Diff limpio entre máquinas.** Si hago fork para desktop y para notebook, el diff queda concentrado en `packages.sh`, sin cambiar shell scripts.
- **Fuente única de verdad.** Todo lo que entra al sistema via gestor de paquetes está en un solo lugar.

## El patrón de los módulos

Cada `<modulo>/install.sh` sigue un esqueleto mínimo. Ejemplo de `git/install.sh`:

```bash
#!/bin/bash

# shellcheck source=helpers.sh
. ../helpers.sh

echo_info "Symlink ~/.gitconfig"
ln -sfT "$HOME/.dotfiles/git/gitconfig" "$HOME/.gitconfig"

echo_done "Git configuration!"
```

La estrategia es siempre la misma: el archivo de config **vive versionado dentro del repo** (`git/gitconfig`), y el install solo crea un symlink en `$HOME` apuntando a él. Eso significa que:

1. Un `git pull` en `~/.dotfiles` actualiza la config en todas las máquinas.
2. Editar `~/.gitconfig` en realidad edita el archivo del repo, entonces los cambios locales quedan rastreables en git.
3. Para "desinstalar" un módulo, basta con borrar el symlink — el archivo original se queda en el repo.

Los módulos que necesitan más que un symlink hacen más — pero mantienen el mismo punto de entrada `install.sh`.

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

Instala oh-my-zsh en modo batch (sin prompt interactivo), clona los tres plugins/themes que uso (autosuggestions, syntax-highlighting, powerlevel10k), hace symlink de mis `.zshrc` y `.p10k.zsh`, y cambia el shell por defecto a zsh — tanto para el usuario como para root.

### `asdf/install.sh` — version manager + lenguajes

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
# ... lo mismo para erlang, elixir, node
```

asdf se clona en `~/.asdf` y hace checkout en el último tag estable. Los plugins de lenguaje se convierten en instalación declarativa via `.tool-versions` versionado.

### `yay/install.sh` — bootstrap del AUR

```bash
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
cd ..
rm -rf yay
```

Este es especial: necesita correr **antes** de `_install aur`, porque `yay` es el helper que va a instalar todo del AUR después. Por eso aparece entre `_install core` y `_install aur` en el flujo principal — es la parte "manual" del bootstrap.

## El flujo completo, del `curl` a la máquina lista

Secuencia de lo que pasa cuando corrés el one-liner en un Arch limpio:

1. `_setup.sh` se baja via curl, valida `git`, clona el repo en `~/.dotfiles`.
2. Empieza el `install.sh` de la raíz.
3. `_update` ajusta el mirror via `geoip` y corre `pacman -Syyu`.
4. `_install core` recorre `${PKG[@]}` instalando todo del oficial.
5. `_symlink` itera por las carpetas y dispara cada `install.sh` (incluido el de `yay`).
6. Cada módulo se ocupa de su propio dotfile: zsh cambia el shell, asdf compila Ruby/Elixir, git linkea `.gitconfig`, etc.
7. Por último, `_install aur` recorre `${AUR[@]}` via `yay`.

Al final, al loguearte, p10k ya está bonito, `ruby -v` retorna 2.7.1, `docker --version` responde, y VS Code está en el menú.

## Reaplicando en una máquina remota

El caso para el que el proyecto fue pensado: VPS Arch nuevo, o Manjaro en un notebook reinstalado. El setup mínimo de mi lado es:

```bash
# 1. Como root, crear usuario y dar sudo
useradd -m -G wheel -s /bin/bash lucio
passwd lucio
visudo  # descomentar %wheel

# 2. Loguearse como el usuario y correr el one-liner
curl -sL https://raw.githubusercontent.com/luciotbc/dotfiles/master/_setup.sh | bash

# 3. Configurar SSH/GPG (instrucciones en el README)
ssh-keygen -t rsa -b 4096 -C "hi@lucio.app"
gpg --default-new-key-algo rsa4096 --gen-key
```

La premisa es que la máquina tiene recursos equivalentes — memoria para compilar Erlang/Ruby via asdf, espacio para el conjunto de paquetes (unos 6–8 GB con todo) y buena red para el initial pull. Si es una máquina más chica, se pueden comentar paquetes en `packages.sh` o módulos pesados antes de correr.

## Lo que cambiaría hoy

Mirando con calma, tres cosas que mejoraría:

- **Idempotencia más fuerte.** Hoy los `install.sh` de los módulos asumen estado limpio. Correrlo dos veces puede fallar en los `git clone` (asdf, oh-my-zsh) porque la carpeta ya existe. Un check del tipo `[ -d ~/.asdf ] || git clone ...` lo resuelve.
- **Versiones de lenguajes fijadas en archivo separado.** Hoy las versiones (Ruby 2.7.1, Node 10.16.0) están hardcodeadas en `asdf/install.sh`. Moverlas a `.tool-versions` y usar solo `asdf install` sin argumentos sería más limpio.
- **Tests en container.** El `.dockerdev/Dockerfile` que existe en el repo da la base para correr el setup completo en un container Arch y validar que termina sin error — bueno para CI.

## Resumen de lo que creo que el proyecto acierta

Lo que más me sirve después de casi un año usando este layout:

- **Modularidad por carpeta** elimina la necesidad de un framework de dotfiles (Dotbot, GNU Stow, chezmoi). Cada módulo es independiente, se lee como código de configuración, no como configuración declarativa abstracta.
- **`packages.sh` separado** hace que "qué está instalado" sea auditable en un solo archivo.
- **Symlink-first** garantiza que mi config vive en git, no en un backup esporádico.
- **Bootstrap remoto** convierte la máquina nueva en un problema de 10 minutos, no de una tarde.

Y principalmente: aprovechar Arch + AUR como repositorio universal de software resuelve lo que en otras distros se convierte en una solución de 4 herramientas (apt + snap + flatpak + AppImage). Eso reduce drásticamente lo que necesito codear en los dotfiles — casi todo es "agregar el nombre del paquete al array".

Repo público en [github.com/luciotbc/dotfiles](https://github.com/luciotbc/dotfiles) si querés hacer fork.
