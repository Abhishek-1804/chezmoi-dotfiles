# dotfiles

Personal machine setup. Two layers:

- **chezmoi** — dotfiles (`~/.zshrc`, `~/.gitconfig`, `~/.config/*`)
- **ansible** — packages, system config, dev tools (idempotent provisioning)

Supports macOS (Apple Silicon) and Ubuntu (desktop + server).

## Layout

```
chezmoi/        # dotfiles, templated per-host
ansible/
  ansible.cfg
  requirements.yml
  inventory/
    hosts.yml
    group_vars/
      all.yml       # shared
      mac.yml       # brew + cask + mas packages
      ubuntu.yml    # apt CLI + GUI packages (GUI gated on `ubuntu_desktop`)
  playbooks/site.yml
  roles/
    bootstrap/            # base tooling, OS-gated
    chezmoi/              # apply dotfiles
    docker/               # ubuntu: docker engine + compose
    lazyvim/              # nvim config bootstrap
    macos_defaults/       # mac: `defaults write` settings
    packages/             # nested via roles_path (see ansible.cfg)
      homebrew/           # mac: brew taps + formulae + casks + mas
      apt/                # ubuntu: CLI + GUI apt packages (GUI gated on hostname)
      external_installers/  # ubuntu: neovim, starship, atuin, lazygit, fastfetch, nerd font
      brew_maintenance/   # mac: brew cleanup (tag: never)
```

## First-time setup

### macOS

```sh
xcode-select --install
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
brew install chezmoi ansible
chezmoi init --apply <github-user>
cd ~/.local/share/chezmoi/ansible
ansible-galaxy collection install -r requirements.yml
ansible-playbook playbooks/site.yml --limit mac --ask-become-pass
```

### Ubuntu

```sh
sudo apt update && sudo apt install -y ansible git
# add target host to ansible/inventory/hosts.yml, then from the control node:
cd ~/.local/share/chezmoi/ansible
ansible-galaxy collection install -r requirements.yml
ansible-playbook playbooks/site.yml --limit ubuntu --ask-become-pass
```

For Ubuntu, **GUI packages install only when "desktop" appears in the inventory hostname**. Name desktops `ubuntu-desktop`, `home-desktop`, etc. Servers — anything without "desktop" in the name — get only the CLI list:

```yaml
ubuntu:
  hosts:
    ubuntu-desktop:               # GUI + CLI
      ansible_host: 10.0.0.133
      ansible_user: akdpubuntu
    my-server:                    # CLI only (no "desktop" in name)
      ansible_host: 10.0.0.50
      ansible_user: ubuntu
```

## Ansible cheat sheet

Run from `ansible/`. The inventory path is set in `ansible.cfg`.

### Inventory checks

```sh
ansible-inventory --list                    # full inventory dump
ansible-inventory --graph                   # tree view of groups/hosts
ansible all -m ping                         # ssh connectivity check
ansible ubuntu -m setup                     # gather + print all facts
ansible ubuntu -m setup -a 'filter=ansible_distribution*'
```

### Ad-hoc commands

```sh
ansible all -m ping                         # built-in ping
ansible ubuntu -a 'uptime'                  # run shell on all ubuntu hosts
ansible ubuntu -b -a 'apt list --upgradable'   # -b = become root
ansible ubuntu -m apt -a 'name=htop state=present' -b
```

### Playbook runs

```sh
ansible-playbook playbooks/site.yml                            # everything
ansible-playbook playbooks/site.yml --limit mac                # one group
ansible-playbook playbooks/site.yml --limit ubuntu-laptop      # one host
ansible-playbook playbooks/site.yml --tags packages            # one role/tag
ansible-playbook playbooks/site.yml --skip-tags lazyvim
ansible-playbook playbooks/site.yml --check --diff             # dry run + show file diffs
ansible-playbook playbooks/site.yml --start-at-task "Install CLI apt packages"
ansible-playbook playbooks/site.yml -vvv                       # verbose (debug ssh issues)
ansible-playbook playbooks/site.yml --ask-become-pass          # prompt for sudo (-K)
ansible-playbook playbooks/site.yml --list-tasks               # show tasks without running
ansible-playbook playbooks/site.yml --list-tags
```

### Useful tag combos in this repo

```sh
# tags wired up in playbooks/site.yml:
#   bootstrap | packages (brew/apt) | external | docker | lazyvim | dotfiles | defaults | maintenance

ansible-playbook playbooks/site.yml --tags dotfiles          # re-apply chezmoi only
ansible-playbook playbooks/site.yml --tags packages          # refresh packages only
ansible-playbook playbooks/site.yml --tags docker --limit ubuntu
ansible-playbook playbooks/site.yml --tags maintenance --limit mac   # brew cleanup (gated by `never`)
```

### Vault (if/when you add secrets)

```sh
ansible-vault create  group_vars/all/vault.yml
ansible-vault edit    group_vars/all/vault.yml
ansible-vault view    group_vars/all/vault.yml
ansible-playbook playbooks/site.yml --ask-vault-pass
```

## chezmoi cheat sheet

```sh
chezmoi status            # what would change
chezmoi diff              # show diff vs target
chezmoi apply             # write to $HOME
chezmoi apply -v -n       # dry run, verbose
chezmoi edit ~/.zshrc     # edit source for a managed file
chezmoi add  ~/.foorc     # start tracking an existing file
chezmoi cd                # cd into source dir
chezmoi update            # git pull + apply
```

## Adding packages

- **mac**: edit `ansible/inventory/group_vars/mac.yml` — `brew_packages`, `cask_packages`, or `mas_packages`.
- **ubuntu CLI** (server-safe): edit `apt_packages_cli` in `ubuntu.yml`.
- **ubuntu GUI** (desktop-only): edit `apt_packages_gui` in `ubuntu.yml`. Skipped unless inventory hostname contains "desktop".
- Things not in apt (neovim, starship, atuin, lazygit, fastfetch, nerd font) live in `roles/external_installers/`.

Re-run: `ansible-playbook playbooks/site.yml --tags packages`.
