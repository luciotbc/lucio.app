---
title: "Tips de macOS: defaults, chflags y limpieza de directorios"
date: 2026-04-30 11:00:00 -0300
updated: 2026-04-30 11:00:00 -0300
tags: [macos, bash, dicas]
excerpt: "Tweaks de macOS que siempre olvido: ajustes del sistema vía defaults write, bloqueo de archivos con chflags, deshabilitar smart quotes globalmente y una limpieza rápida de directorios."
lang: es
ref: dicas-de-macos-defaults-chflags-e-limpeza
---

Compilando aquí algunas anotaciones sueltas que tengo sobre macOS — ajustes que hago siempre que configuro una máquina nueva o que necesito aplicar de nuevo después de una actualización.

## Limpiar basura de directorios

Cuando descargás un zip o recibís una carpeta compartida, suele venir acompañada de una serie de archivos auxiliares que no interesan — `.DS_Store` del Finder, `Thumbs.db` de Windows, miniaturas e índices de cámaras, etc. Para limpiar todo de una vez:

```bash
find . -name '.DS_Store' -type f -delete
find . -name 'Thumbs.db' -type f -delete
find . -name "*.THM" -type f -delete
find . -name "*.ini" -type f -delete
find . -name "*.LRV" -type f -delete
find . -name "*.db" -type f -delete
find . -name "*.nomedia" -type f -delete
```

Después de eso, aprovechá para eliminar directorios vacíos — primero listar para verificar, luego borrar:

```bash
# Listar
find . -type d -empty
# Borrar
find . -type d -empty -delete
```

Y un truco que siempre uso para entender qué hay dentro de una carpeta antes de tocarla: listar todas las extensiones de archivo distintas presentes:

```bash
find . -type f -name '*.*' | sed 's|.*\.||' | sort -u
```

Útil para decidir qué limpiar o qué procesar en lote.

## Ajustes del sistema con `defaults write`

`defaults` es el utilitario de línea de comandos que lee y escribe las preferencias de macOS. Buena parte de las configuraciones que encontrarías escondidas en Preferencias (o que ni aparecen en la UI) son accesibles desde aquí.

### Cmd+Tab en todos los monitores

Por defecto, el selector de aplicaciones del Cmd+Tab solo aparece en el monitor principal. Si usás múltiples displays, esto es molesto — estás enfocado en el monitor secundario, presionás Cmd+Tab y el selector aparece del otro lado del escritorio. Para que aparezca en el monitor donde está el cursor:

```bash
defaults write com.apple.Dock appswitcher-all-displays -bool true
killall Dock
```

`killall Dock` reinicia el Dock para que la configuración se aplique.

### Deshabilitar smart quotes y smart dashes

macOS reemplaza automáticamente `"` por `"` `"` y `--` por `—` mientras escribís. Esto es genial para escribir en el Pages, y pésimo para escribir código o markdown en cualquier otro lugar. Para desactivarlo globalmente:

```bash
defaults write NSGlobalDomain NSAutomaticQuoteSubstitutionEnabled -bool false
defaults write NSGlobalDomain NSAutomaticDashSubstitutionEnabled -bool false
defaults write com.apple.TextEdit SmartQuotes -bool false
defaults write com.apple.TextEdit SmartDashes -bool false
```

`NSGlobalDomain` afecta el comportamiento por defecto de todas las apps — pero algunas apps ignoran este global y mantienen su propia configuración. Para forzarlo en todas las apps registradas en el sistema, podés recorrerlas:

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

`osascript -e "app id \"$d\""` filtra solo los dominios que corresponden a apps reales — sin esto, estarías escribiendo configuración en dominios de sistema que no tienen app asociada. Dejé las otras líneas comentadas porque yo mismo solo deshabilito las smart quotes; pero podés activarlas según el gusto.

## Bloquear el `/etc/hosts` con `chflags`

Varios malwares y hasta algunos instaladores de software tienen la costumbre de modificar el `/etc/hosts`. Si querés bloquear el archivo de modo que ni `sudo` pueda editarlo sin desbloquearlo antes:

```bash
sudo chflags uchg /etc/hosts && sudo chflags schg /etc/hosts
```

- `uchg` → user immutable flag (cualquier usuario queda impedido de modificar)
- `schg` → system immutable flag (ni root modifica sin desbloquear; solo removible en single-user mode)

Para desbloquear cuando necesitás editar:

```bash
sudo chflags nouchg /etc/hosts && sudo chflags noschg /etc/hosts
```

Vale para cualquier archivo, no solo `hosts`. Es una capa de protección extra contra modificación accidental o maliciosa.

## Deshabilitar que Music.app se abra solo

Este era el que más me molestaba: presionar play en los auriculares y Music.app se abría de la nada, aunque estuviera escuchando Spotify o YouTube. Intenté varias aproximaciones vía `defaults`, `launchctl`, scripts de Automator — nada funcionó de forma confiable en versiones recientes de macOS. Apple bloqueó cada vector que existía.

La solución que funcionó para mí fue instalar **BetterTouchTool** y mapear las teclas de medios a un no-op (o a la app que realmente quiero controlar). No es una solución nativa, pero es la única que sobrevivió a las actualizaciones del sistema.

Anotando esto aquí más como un recordatorio: no vale la pena gastar tiempo buscando el `defaults write` mágico que deshabilita esto. Ya no existe. Usá una herramienta externa.
