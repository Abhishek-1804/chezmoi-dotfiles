# Dotfiles

Personal macOS configuration managed with [chezmoi](https://www.chezmoi.io/).

## Quick Start

### Fresh macOS Install

```bash
sh -c "$(curl -fsLS get.chezmoi.io)" -- init --apply <your-github-username>
```

This will:
1. Install chezmoi
2. Clone this repository
3. Run setup scripts in order
4. Apply all dotfiles

### Existing Setup

If chezmoi is already installed:

```bash
chezmoi init --apply <your-github-username>
```

## What's Included

### Applications & Tools

**Development:**
- Languages: Node.js, Python, Go, Rust, Java (Gradle)
- Editors: Neovim (LazyVim)
- Version Control: Git, GitHub CLI
- Package Managers: pnpm, uv, rustup

**CLI Tools:**
- Shell: Zsh with Oh-My-Zsh
- Modern replacements: `bat`, `eza`, `fd`, `ripgrep`, `fzf`
- Terminal: Ghostty, Zellij (multiplexer)
- Dev utilities: `lazygit`, `lazydocker`, `k9s`, `atuin`
- System monitoring: `btop`, `duf`, `fastfetch`

**GUI Applications:**
- Browsers: Arc, Chrome, Vivaldi
- IDEs: Cursor, Windsurf
- Containers: OrbStack, UTM
- Productivity: Raycast, Notion
- AI Tools: Claude, Claude Code, ChatGPT, LM Studio

**Cloud & DevOps:**
- Container tools: Podman, kubectl, k9s, Minikube
- Cloud CLIs: AWS CLI
- Utilities: opencode

### Configurations

- **Zsh** (`~/.zshrc`): Shell configuration with Oh-My-Zsh
- **Neovim** (`~/.config/nvim/`): LazyVim distribution with custom plugins
- **Starship** (`~/.config/starship.toml`): Cross-shell prompt
- **Zellij** (`~/.config/zellij/config.kdl`): Terminal multiplexer
- **Atuin** (`~/.config/atuin/config.toml`): Shell history sync
- **Ghostty** (`~/.config/ghostty/config`): Terminal emulator

### macOS System Defaults

- Dark mode enabled
- Finder: Column view by default
- Fast key repeat rate
- Touch ID for sudo
- Guest account disabled

## Setup Process

Scripts run in this order before dotfiles are applied:

### 1. System Setup (`run_once_before_01-setup-macos.sh.tmpl`)
Runs once on first install:
- Installs Homebrew (if not present)
- Configures macOS system defaults
- Installs Oh-My-Zsh with plugins:
  - zsh-syntax-highlighting
  - zsh-autosuggestions

### 2. Package Installation (`run_onchange_before_02-install-packages.sh.tmpl`)
Runs when package list changes:
- Installs/updates all Homebrew packages
- Removes packages not in the list (declarative)

### 3. LazyVim Installation (`run_onchange_before_03-install-lazyvim.sh.tmpl`)
Runs when script changes or if `init.lua` is missing:
- Clones LazyVim starter
- Your custom configs overlay on top

### 4. Dotfiles Applied
Your configuration files are applied to `~/`

## Manual Steps

After setup completes, you need to manually:

1. **Remap Caps Lock to Control:**
   - System Settings > Keyboard > Keyboard Shortcuts > Modifier Keys
   - Select your keyboard
   - Set Caps Lock → Control

2. **Configure applications:**
   - Import browser settings
   - Sign into cloud services
   - Configure Raycast/AeroSpace shortcuts

3. **Restart or logout** for all system settings to take effect

## Usage

### Update dotfiles

```bash
chezmoi update
```

### Edit a file

```bash
chezmoi edit ~/.zshrc
```

### Apply changes

```bash
chezmoi apply
```

### Add a new file

```bash
chezmoi add ~/.config/newapp/config.toml
```

### Check what would change

```bash
chezmoi diff
```

## Customization

### Adding Packages

Edit `run_onchange_before_02-install-packages.sh.tmpl` and modify the Brewfile section.

Run `chezmoi apply` to install new packages and remove unlisted ones.

### Modifying Configs

```bash
# Edit in the source repository
chezmoi edit ~/.config/nvim/lua/plugins/custom.lua

# Or edit directly and add to chezmoi
vim ~/.config/nvim/lua/plugins/custom.lua
chezmoi add ~/.config/nvim/lua/plugins/custom.lua
```

### Private/Encrypted Files

Files prefixed with `private_` are encrypted (starship.toml, zellij config).

## Structure

```
~/.local/share/chezmoi/
├── .chezmoiignore                           # Files to not deploy
├── README.md                                 # This file
├── dot_zshrc                                 # ~/.zshrc
├── dot_config/
│   ├── nvim/                                 # Neovim configs
│   ├── private_starship.toml                 # Encrypted
│   ├── private_zellij/                       # Encrypted
│   ├── atuin/
│   └── ghostty/
└── run_*_before_*.sh.tmpl                    # Setup scripts
```

## Notes

- All `run_once_*` scripts execute only on first install
- All `run_onchange_*` scripts re-run when their content changes
- Scripts with `before` run before dotfiles are applied
- Package management is declarative - unlisted packages are removed

## License

MIT
