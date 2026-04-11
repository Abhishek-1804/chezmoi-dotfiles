# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A [chezmoi](https://chezmoi.io) dotfiles repository for macOS. Changes here are applied to the home directory by running `chezmoi apply`.

## Applying changes

```bash
chezmoi apply          # apply all pending changes
chezmoi apply -v       # verbose — shows what would change
chezmoi diff           # preview changes without applying
chezmoi status         # show which files differ from source
```

## File naming conventions (chezmoi source state)

| Prefix/suffix | Meaning |
|---|---|
| `dot_` | Maps to a dotfile (e.g. `dot_zshrc` → `~/.zshrc`) |
| `.tmpl` suffix | Go template — processed before applying |
| `run_once_before_*` | Script runs once ever (tracked in `~/.config/chezmoi/chezmoistate.boltdb`) |
| `run_onchange_*` | Script runs whenever the file's content changes |
| `private_` | Sets file permissions to 0600 |

## Template variables

Template data comes from two places:

- `~/.config/chezmoi/chezmoi.toml` — personal data: `.name`, `.email`
- `.chezmoidata/packages.yaml` — declarative package list: `.packages.darwin.{taps,brews,casks,mas}`

Templates use Go's `text/template` syntax. The `.chezmoi.os` variable is `"darwin"` on macOS — all scripts are guarded with `{{ if eq .chezmoi.os "darwin" }}`.

## Package management

Packages are declared in `.chezmoidata/packages.yaml` under `packages.darwin`. The `run_onchange_02_darwin_install_packages.sh.tmpl` script runs `brew bundle --cleanup --upgrade`, meaning:

- Adding a package to the yaml → installs it next `chezmoi apply`
- Removing a package → uninstalls it next `chezmoi apply` (via `--cleanup`)

## Scripts overview

| File | When it runs | What it does |
|---|---|---|
| `run_once_before_01_setup_environment.sh.tmpl` | Once on first apply | Xcode CLI tools, Homebrew, macOS defaults, Oh-My-Zsh, LazyVim |
| `run_onchange_02_darwin_install_packages.sh.tmpl` | On package list change | `brew bundle --cleanup --upgrade` |
| `run_onchange_03_darwin_login_items.sh.tmpl` | On script change | Syncs macOS login items via AppleScript |

## External dependencies

`.chezmoiexternal.toml` clones Oh-My-Zsh plugins (`zsh-syntax-highlighting`, `zsh-autosuggestions`) as git repos, refreshed every 168 hours.

## App configs managed

Under `dot_config/`: `aerospace` (tiling WM), `atuin` (shell history), `ghostty` (terminal), `nvim` (LazyVim), `starship` (prompt).
