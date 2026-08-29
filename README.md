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
  inventory/hosts.yml       # one entry per machine, all ansible_connection: local
  hosts/                    # one self-contained tree per host
    mac/
      playbook.yml
      vars.yml              # brew + cask + mas packages
      roles/                # bootstrap, homebrew, lazyvim, chezmoi,
                            # macos_defaults, brew_maintenance
    ubuntu-desktop/
      playbook.yml
      vars.yml              # apt packages (CLI + GUI)
      roles/                # bootstrap, apt, external_installers,
                            # docker, lazyvim, chezmoi
    ubuntu-server/
      playbook.yml
      vars.yml              # apt packages (CLI only)
      roles/                # same set as ubuntu-desktop, minus the nerd font
```

Roles are **duplicated per host on purpose**. Each `hosts/<name>/` tree is standalone:
no OS gating, no hostname conditionals, no shared roles. Editing one host cannot
break another. The cost is that a fix worth having everywhere must be applied in
each tree.

## First-time setup

### macOS

```sh
xcode-select --install
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
brew install chezmoi ansible
chezmoi init --apply <github-user>
cd ~/dotfiles/ansible
ansible-galaxy collection install -r requirements.yml
ansible-playbook hosts/mac/playbook.yml --ask-become-pass
```

### Ubuntu

Run on the machine itself — pick the tree that matches what the box is.

```sh
sudo apt update && sudo apt install -y ansible git
cd ~/dotfiles/ansible
ansible-galaxy collection install -r requirements.yml

# desktop (GUI packages + nerd font)
ansible-playbook hosts/ubuntu-desktop/playbook.yml --ask-become-pass

# server (CLI only)
ansible-playbook hosts/ubuntu-server/playbook.yml --ask-become-pass
```

### Adding a new machine

Copy the closest existing tree, then register the host:

```sh
cp -R hosts/ubuntu-server hosts/ubuntu-nas
# edit hosts/ubuntu-nas/playbook.yml  -> hosts: ubuntu-nas
# edit hosts/ubuntu-nas/vars.yml      -> its package list
```

```yaml
# inventory/hosts.yml
all:
  hosts:
    ubuntu-nas:
      ansible_connection: local
      ansible_python_interpreter: /usr/bin/python3
```

## Follow these steps manually

> "Not every step deserves automation. One-time setup is productive procrastination dressed up as engineering — sometimes `curl`-ing a binary beats an hour spent making it declarative."

### macOS

- Add apps to **Login Items** (System Settings → General → Login Items).

### Ubuntu

- Enable SSH: `sudo apt install -y openssh-server && sudo systemctl enable --now ssh`.
- Install Tailscale if needed: `curl -fsSL https://tailscale.com/install.sh | sh && sudo tailscale up`.
- On `ubuntu-desktop` used as a server (laptop lid closed, on AC): enable TLP for battery longevity — `sudo systemctl enable --now tlp tlp-sleep` (tlp + tlp-rdw installed via `apt_packages`).

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

Run from `ansible/`. Inventory path set in `ansible.cfg`. There is no combined
playbook — each host is its own entry point.

```sh
ansible-playbook hosts/mac/playbook.yml                    # provision this mac
ansible-playbook hosts/mac/playbook.yml --tags packages    # one tag
ansible-playbook hosts/mac/playbook.yml --check --diff     # dry run
ansible-playbook hosts/mac/playbook.yml --ask-become-pass  # prompt for sudo
ansible-playbook hosts/mac/playbook.yml --tags maintenance # brew cleanup (tag: never)
```

Tags wired in every `playbook.yml`: `bootstrap | packages | external | docker | lazyvim | dotfiles | defaults | maintenance`.

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

Every host owns its own list in `hosts/<name>/vars.yml`:

- **mac** — `brew_packages`, `cask_packages`, `mas_packages` in `hosts/mac/vars.yml`.
- **ubuntu-desktop** — `apt_packages` in `hosts/ubuntu-desktop/vars.yml` (GUI packages
  just go in the same list; nothing is gated).
- **ubuntu-server** — `apt_packages` in `hosts/ubuntu-server/vars.yml`.

Want a package on both ubuntu hosts? Add it to both files.

Things not in apt (neovim, starship, atuin, lazygit, fastfetch, k8s tools, nerd font)
live in that host's `roles/external_installers/`, with pinned versions in its
`defaults/main.yml`.

Re-run: `ansible-playbook hosts/<name>/playbook.yml --tags packages`.
