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

**Split per host, not per OS.** Every machine gets a self-contained tree under
`ansible/hosts/<name>/`:

```
ansible/
  ansible.cfg
  inventory/hosts.yml        # one entry per machine, all ansible_connection: local
  hosts/
    mac/            playbook.yml  vars.yml  roles/
    ubuntu-desktop/ playbook.yml  vars.yml  roles/
    ubuntu-server/  playbook.yml  vars.yml  roles/
```

There is **no shared `roles/` directory and no `site.yml`**. Ansible resolves roles
from the `roles/` dir adjacent to the playbook, so each tree loads only its own
copies. `ansible.cfg` has no `roles_path` — adding one would reintroduce sharing.

### The duplication is deliberate

`lazyvim`, `chezmoi`, `docker`, `bootstrap`, and `external_installers` exist as
independent copies in multiple trees. This was chosen over shared roles so that
each host can be edited without regression risk elsewhere. Consequences:

- **Never** refactor duplicated roles back into a shared location.
- **Never** add `when: ansible_facts['os_family'] == ...` or hostname conditionals.
  A tree already knows what OS it is: `hosts/mac/roles/bootstrap` is macOS-only,
  `hosts/ubuntu-*/roles/bootstrap` is Debian-only, both with the gate stripped.
- A fix worth having everywhere must be applied in each tree by hand. When asked to
  change shared behaviour, ask which hosts it applies to, or apply to all and say so.

### Per-host role sets

| Host | Roles |
|---|---|
| `mac` | bootstrap, homebrew, lazyvim, chezmoi, macos_defaults, brew_maintenance |
| `ubuntu-desktop` | bootstrap, apt, external_installers (incl. nerd font), docker, lazyvim, chezmoi |
| `ubuntu-server` | bootstrap, apt, external_installers, docker, lazyvim, chezmoi |

The old `apt_cli`/`apt_gui` and `external_installers_cli`/`_gui` splits are gone —
the CLI/GUI distinction is now just which host tree a package is listed in.

### Package sources (where to add things)

| Host | File | Lists |
|---|---|---|
| mac | `ansible/hosts/mac/vars.yml` | `brew_taps`, `brew_packages`, `cask_packages`, `brew_services`, `mas_packages` |
| ubuntu-desktop | `ansible/hosts/ubuntu-desktop/vars.yml` | `apt_packages` (includes GUI packages) |
| ubuntu-server | `ansible/hosts/ubuntu-server/vars.yml` | `apt_packages` (CLI only) |

Vars are loaded via `vars_files: [vars.yml]` in each playbook, resolved relative to
the playbook. `inventory/group_vars/` no longer exists; `user_name`/`user_email` are
repeated in each host's `vars.yml`.

### Why `external_installers` exists (ubuntu only)

Installs tools from upstream where apt is missing, stale, or unsuitable: neovim,
starship, atuin, lazygit, lazydocker, fastfetch, kind, kubectl, helm, helmfile, k9s,
uv, claude-code, plus bat/fd alias symlinks. `ubuntu-desktop` additionally installs
the Nerd Font (only useful where a terminal renders it).

Why upstream instead of apt:

- apt neovim is too old for LazyVim
- apt chezmoi lacks template features (chezmoi installed via official script in `bootstrap`)
- fastfetch isn't in 24.04 apt
- starship/atuin/lazygit/lazydocker/nerd-fonts/k8s tools aren't packaged

When adding a tool to an Ubuntu host: prefer apt (`apt_packages` in that host's
`vars.yml`). Fall back to `hosts/<name>/roles/external_installers/tasks/<tool>.yml`
only if apt is missing or stale, and document the reason in the task file. Pinned
versions live in that role's `defaults/main.yml`.

### Adding a new machine

Copy the closest existing tree (`cp -R hosts/ubuntu-server hosts/ubuntu-nas`), change
`hosts:` in its `playbook.yml`, trim its `vars.yml`, and add the host to
`inventory/hosts.yml`. Nothing else references it.

### Tag conventions

Every role in every `playbook.yml` has tags: `bootstrap | packages | apt | brew |
external | docker | lazyvim | nvim | dotfiles | chezmoi | defaults | maintenance`.
Two non-obvious ones:

- `--tags packages` — refreshes brew on mac, apt + external_installers on ubuntu
- `brew_maintenance` (mac only) is tagged `never` — runs only with `--tags maintenance`

## Common commands

Run from `ansible/` (the inventory path is set in `ansible.cfg`). Each host tree is
its own entry point — there is no playbook that runs them all.

```sh
# first-time deps
ansible-galaxy collection install -r requirements.yml

# provision the machine you are on
ansible-playbook hosts/mac/playbook.yml --ask-become-pass
ansible-playbook hosts/ubuntu-desktop/playbook.yml --ask-become-pass
ansible-playbook hosts/ubuntu-server/playbook.yml --ask-become-pass

# scoped runs
ansible-playbook hosts/mac/playbook.yml --tags dotfiles      # re-apply chezmoi only
ansible-playbook hosts/mac/playbook.yml --tags packages      # refresh packages only
ansible-playbook hosts/mac/playbook.yml --tags maintenance   # brew cleanup (tag: never)

# dry run with diffs
ansible-playbook hosts/mac/playbook.yml --check --diff

# inspect
ansible-playbook hosts/mac/playbook.yml --syntax-check
ansible-playbook hosts/mac/playbook.yml --list-tasks
ansible-inventory --graph

# chezmoi (outside ansible)
chezmoi diff
chezmoi apply -v -n        # dry run
chezmoi edit ~/.zshrc      # edit source via $HOME path
```

## Gotchas

- **apt binary name drift**: on Ubuntu `bat` is `batcat`, `fd` is `fdfind`. Each ubuntu
  tree's `roles/external_installers/tasks/aliases.yml` symlinks them back to the
  expected names — do not "fix" callers to use the apt names.
- **chezmoi from apt is broken** for this repo's templates. The `bootstrap` role installs
  chezmoi via the official script; do not add it to `apt_packages`.
- **All hosts are `ansible_connection: local`.** The playbooks provision the machine they
  run on; there is no SSH/control-node path. Adding `ansible_host`/`ansible_user` to
  reprovision a box remotely is a deliberate change, not a fill-in-the-blank.
- The `host_key_checking = False` in `ansible.cfg` is intentional for fresh-provisioned
  hosts; do not remove without replacing with a known_hosts strategy.
