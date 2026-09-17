---
title: "macOS tips: defaults, chflags and directory cleanup"
date: 2026-04-30 11:00:00 -0300
updated: 2026-04-30 11:00:00 -0300
tags: [macos, bash, dicas]
excerpt: "macOS tweaks I always forget: system adjustments via defaults write, file locking with chflags, disabling smart quotes globally, and a quick directory cleanup."
lang: en
ref: dicas-de-macos-defaults-chflags-e-limpeza
---

Compiling here some loose notes I have on macOS — adjustments I make every time I set up a new machine or need to apply again after an update.

## Cleaning up directory junk

When you download a zip or receive a shared folder, it usually comes with a bunch of auxiliary files that you don't want — Finder's `.DS_Store`, Windows' `Thumbs.db`, camera thumbnails and indexes, etc. To clean everything up at once:

```bash
find . -name '.DS_Store' -type f -delete
find . -name 'Thumbs.db' -type f -delete
find . -name "*.THM" -type f -delete
find . -name "*.ini" -type f -delete
find . -name "*.LRV" -type f -delete
find . -name "*.db" -type f -delete
find . -name "*.nomedia" -type f -delete
```

After that, take the opportunity to remove empty directories — first list to verify, then delete:

```bash
# List
find . -type d -empty
# Delete
find . -type d -empty -delete
```

And a trick I always use to understand what's inside a folder before touching it: list all distinct file extensions present:

```bash
find . -type f -name '*.*' | sed 's|.*\.||' | sort -u
```

Useful for deciding what to clean up or what to process in bulk.

## System adjustments with `defaults write`

`defaults` is the command-line utility that reads and writes macOS preferences. Most of the settings you'd find buried in Preferences (or that don't even show up in the UI) are accessible here.

### Cmd+Tab on all monitors

By default, the Cmd+Tab app switcher only appears on the primary monitor. If you use multiple displays, this is annoying — you're focused on the secondary monitor, press Cmd+Tab and the switcher appears on the other side of the desk. To make it appear on the monitor where the cursor is:

```bash
defaults write com.apple.Dock appswitcher-all-displays -bool true
killall Dock
```

`killall Dock` restarts the Dock so the setting takes effect.

### Disable smart quotes and smart dashes

macOS automatically replaces `"` with `"` `"` and `--` with `—` while you type. This is great for writing in Portuguese in Pages, and terrible for writing code or markdown anywhere else. To disable globally:

```bash
defaults write NSGlobalDomain NSAutomaticQuoteSubstitutionEnabled -bool false
defaults write NSGlobalDomain NSAutomaticDashSubstitutionEnabled -bool false
defaults write com.apple.TextEdit SmartQuotes -bool false
defaults write com.apple.TextEdit SmartDashes -bool false
```

`NSGlobalDomain` affects the default behavior of all apps — but some apps ignore this global and keep their own setting. To force it across all apps registered in the system, you can sweep through them:

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

`osascript -e "app id \"$d\""` filters only domains that correspond to real apps — without this, you'd be writing configuration to system domains that have no associated app. I left the other lines commented because I only disable smart quotes myself; but you can enable them as needed.

## Locking `/etc/hosts` with `chflags`

Several malware programs and even some software installers have the habit of modifying `/etc/hosts`. If you want to lock the file so that even `sudo` can't edit it without unlocking first:

```bash
sudo chflags uchg /etc/hosts && sudo chflags schg /etc/hosts
```

- `uchg` → user immutable flag (any user is prevented from modifying it)
- `schg` → system immutable flag (not even root can modify it without unlocking; only removable in single-user mode)

To unlock when you need to edit:

```bash
sudo chflags nouchg /etc/hosts && sudo chflags noschg /etc/hosts
```

Works for any file, not just `hosts`. It's an extra layer of protection against accidental or malicious modification.

## Disabling Music.app from opening on its own

This was the one that annoyed me most: pressing play on the headset and Music.app would open out of nowhere, even if I was listening to Spotify or YouTube. I tried various approaches via `defaults`, `launchctl`, Automator scripts — nothing worked reliably in recent versions of macOS. Apple has blocked every vector that existed.

The solution that worked for me was installing **BetterTouchTool** and mapping the media keys to a no-op (or to the app I actually want to control). It's not a native solution, but it's the only one that survived system updates.

Noting this here more as a reminder: it's not worth spending time searching for the magic `defaults write` that disables this. It no longer exists. Use an external tool.
