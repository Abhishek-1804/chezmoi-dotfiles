# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Personal machine setup with two cooperating layers:

- **`chezmoi/`** — dotfiles source tree (`~/.zshrc`, `~/.gitconfig.tmpl`, `~/.config/*`). Applied to `$HOME` by chezmoi.
- **`ansible/`** — provisioning (packages, system config, dev tools) for macOS and Ubuntu.

The two are linked: the `chezmoi` Ansible role runs `chezmoi apply` at the end of provisioning, so a full `ansible-playbook` run installs packages **and** lays down dotfiles in one shot.

## Critical layout detail: `.chezmoiroot`

`.chezmoiroot` at the repo root contains `chezmoi`, which tells chezmoi its source directory is the `chezmoi/` **subfolder**, not the repo root. Without this, chezmoi would try to apply `ansible/` and `README.md` as dotfiles. Any new top-level files (docs, CI config, etc.) live safely outside the chezmoi source tree because of this.

## Ansible architecture

Single playbook (`ansible/playbooks/site.yml`) drives two host groups defined in `ansible/inventory/hosts.yml`:

- **`mac`** — `localhost`, `ansible_connection: local`
- **`ubuntu`** — remote SSH hosts (currently `ubuntu-laptop` at `10.0.0.133`)

Each group has its own role pipeline. Roles common to both (`bootstrap`, `lazyvim`, `chezmoi`) are OS-gated internally.

### Package sources (where to add things)

| Platform | File | Lists |
|---|---|---|
| mac | `ansible/inventory/group_vars/mac.yml` | `brew_taps`, `brew_packages`, `cask_packages`, `brew_services`, `mas_packages` |
| ubuntu | `ansible/inventory/group_vars/ubuntu.yml` | `apt_packages_cli` (server-safe), `apt_packages_gui` (desktop-only) |

Ubuntu CLI vs GUI split is two separate roles: `apt_cli` (all ubuntu hosts) and `apt_gui` (desktop only). `site.yml` assigns `apt_cli` to the `ubuntu` group and `apt_gui` to `ubuntu-desktop` explicitly — no hostname-based conditionals.

### Role layout

Package-related roles live under `roles/packages/` (apt_cli, apt_gui, homebrew, external_installers_cli, external_installers_gui, brew_maintenance). `ansible.cfg` adds `roles/packages` to `roles_path`, so site.yml references them by short name (`role: apt_cli`). Non-package roles (bootstrap, chezmoi, docker, lazyvim, macos_defaults) stay at the top level of `roles/`. Note this nesting is **not standard** Ansible convention (convention is flat `roles/`); it's a deliberate organization choice for this repo.

### Why `external_installers_cli` / `external_installers_gui` exist (ubuntu only)

These roles install tools from upstream (not apt) where apt is missing, stale, or unsuitable. Split mirrors the `apt_cli` / `apt_gui` pattern: `_cli` runs on every ubuntu host, `_gui` only on `ubuntu-desktop`.

- `external_installers_cli` — neovim, starship, atuin, lazygit, lazydocker, fastfetch, kind, kubectl, helm, helmfile, k9s, uv, claude-code, plus bat/fd alias symlinks
- `external_installers_gui` — Nerd Font (only useful on host with rendering terminal)

Why upstream instead of apt:

- apt neovim is too old for LazyVim
- apt chezmoi lacks template features (chezmoi installed via official script in `bootstrap`)
- fastfetch isn't in 24.04 apt
- starship/atuin/lazygit/lazydocker/nerd-fonts/k8s tools aren't packaged

When adding a tool to Ubuntu: prefer apt (`apt_packages_cli`/`apt_packages_gui`). Fall back to `external_installers_cli/tasks/<tool>.yml` (or `_gui` for desktop-only tools like terminals/fonts) only if apt is missing or stale. Document the reason in the task file.

### Tag conventions

Every role in `site.yml` has tags. Two non-obvious ones:

- `--tags packages` — refreshes brew on mac, apt + external_installers on ubuntu
- `brew_maintenance` role is tagged `never` — runs only with `--tags maintenance`

### Bootstrap gating

`roles/bootstrap/tasks/main.yml` dispatches by `ansible_facts['os_family']` to `darwin.yml` or `linux.yml`. Same pattern is the right one if any other role needs OS-specific paths.

## Common commands

Run from `ansible/` (the inventory path is set in `ansible.cfg`).

```sh
# first-time deps
ansible-galaxy collection install -r requirements.yml

# full run
ansible-playbook playbooks/site.yml --ask-become-pass

# scoped runs
ansible-playbook playbooks/site.yml --limit mac
ansible-playbook playbooks/site.yml --limit ubuntu
ansible-playbook playbooks/site.yml --tags dotfiles        # re-apply chezmoi only
ansible-playbook playbooks/site.yml --tags packages        # refresh packages only

# dry run with diffs
ansible-playbook playbooks/site.yml --check --diff

# debug ssh / connectivity
ansible all -m ping
ansible-inventory --graph
ansible-playbook playbooks/site.yml -vvv

# chezmoi (outside ansible)
chezmoi diff
chezmoi apply -v -n        # dry run
chezmoi edit ~/.zshrc      # edit source via $HOME path
```

## Gotchas

- **apt binary name drift**: on Ubuntu `bat` is `batcat`, `fd` is `fdfind`. `roles/external_installers/tasks/aliases.yml` symlinks them back to the expected names — do not "fix" callers to use the apt names.
- **chezmoi from apt is broken** for this repo's templates. The `bootstrap` role installs chezmoi via the official script; do not add it to `apt_packages_cli`.
- **`ansible_user` on Ubuntu** is `akdpubuntu` (not `ubuntu` or the current shell user). New Ubuntu hosts need their own user set in `inventory/hosts.yml`.
- The `host_key_checking = False` in `ansible.cfg` is intentional for fresh-provisioned hosts; do not remove without replacing with a known_hosts strategy.
