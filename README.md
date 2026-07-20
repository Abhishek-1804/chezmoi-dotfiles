# dotfiles

Personal machine setup. This repo manages **only three things**:

1. **Packages** — everything installed via brew/apt/external installers
2. **System config** — OS-level settings (`defaults write`, etc.)
3. **Dotfiles** — `~/.zshrc`, `~/.gitconfig`, `~/.config/*`

Anything outside those three is intentionally out of scope.

Two layers:

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
cd ~/dotfiles/ansible
ansible-galaxy collection install -r requirements.yml
ansible-playbook playbooks/site.yml --limit mac --ask-become-pass
```

### Ubuntu

```sh
sudo apt update && sudo apt install -y ansible git
# add target host to ansible/inventory/hosts.yml, then from the control node:
cd ~/dotfiles/ansible
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

## Follow these steps manually

> "Not every step deserves automation. One-time setup is productive procrastination dressed up as engineering — sometimes `curl`-ing a binary beats an hour spent making it declarative."

### macOS

- Add apps to **Login Items** (System Settings → General → Login Items).

### Ubuntu

- Enable SSH: `sudo apt install -y openssh-server && sudo systemctl enable --now ssh`.
- Install Tailscale if needed: `curl -fsSL https://tailscale.com/install.sh | sh && sudo tailscale up`.
- On `ubuntu-desktop` used as a server (laptop lid closed, on AC): enable TLP for battery longevity — `sudo systemctl enable --now tlp tlp-sleep` (tlp + tlp-rdw installed via `apt_packages_cli`).

### General

- Authenticate to GitHub via SSH:

  ```sh
  ssh-keygen -t ed25519 -C "your_email@example.com"
  eval "$(ssh-agent -s)"
  ssh-add ~/.ssh/id_ed25519
  cat ~/.ssh/id_ed25519.pub        # paste into github.com/settings/keys
  ssh -T git@github.com            # verify
  ```

## Ansible cheat sheet

Run from `ansible/`. Inventory path set in `ansible.cfg`.

```sh
ansible-playbook playbooks/site.yml                       # everything
ansible-playbook playbooks/site.yml --limit mac           # one group
ansible-playbook playbooks/site.yml --tags packages       # one tag
ansible-playbook playbooks/site.yml --check --diff        # dry run
ansible-playbook playbooks/site.yml --ask-become-pass     # prompt for sudo
```

Tags wired in `playbooks/site.yml`: `bootstrap | packages | external | docker | lazyvim | dotfiles | defaults | maintenance`.

Full reference: [docs.ansible.com](https://docs.ansible.com/ansible/latest/).

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
