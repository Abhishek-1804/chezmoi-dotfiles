# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a macOS dotfiles repository managed with [chezmoi](https://www.chezmoi.io/). It contains personal system configuration including shell setup, application configs, and automated macOS installation scripts.

## Key Concepts

### Chezmoi Naming Conventions
- `dot_` prefix → `.` (e.g., `dot_zshrc` becomes `~/.zshrc`)
- `private_` prefix → encrypted files (requires encryption setup)
- `run_once_before_*` → runs once on first install, before dotfiles applied
- `run_onchange_before_*` → re-runs when file content changes, before dotfiles applied
- `.tmpl` suffix → template files (chezmoi processes these with Go templates)

### Script Execution Order
1. `run_once_before_01-setup-macos.sh.tmpl` - Initial system setup
2. `run_onchange_before_02-install-packages.sh.tmpl` - Declarative package management
3. `run_onchange_before_03-install-lazyvim.sh.tmpl` - Neovim setup
4. Dotfiles are applied to `~/`

## Common Commands

### Testing and Validation
```bash
# Preview what would change without applying
chezmoi diff

# Validate templates compile correctly
chezmoi execute-template < filename.tmpl

# Apply changes (does NOT run if working in source directory)
chezmoi apply

# Update from remote repository
chezmoi update
```

### Development Workflow
```bash
# Edit a file managed by chezmoi
chezmoi edit ~/.zshrc

# Add a new file to chezmoi
chezmoi add ~/.config/newapp/config.toml

# View the source state
chezmoi cd

# Return to previous directory
exit
```

### Package Management
Edit `run_onchange_before_02-install-packages.sh.tmpl` to modify the Brewfile section. The script is DECLARATIVE - packages not listed will be removed on next `chezmoi apply`.

## Important Constraints

### File Operations
- **NEVER directly edit files in `~/`** - always use `chezmoi edit` or edit source files in `~/.local/share/chezmoi/`
- **Encrypted files** (`private_*`) require chezmoi encryption to be configured to read/modify
- Files listed in `.chezmoiignore` (currently just README.md) are not deployed to `~/`

### Package Management
- The Brewfile in `run_onchange_before_02-install-packages.sh.tmpl` is DECLARATIVE
- Adding a package: Add line to appropriate section in the Brewfile
- Removing a package: Delete line from Brewfile (will be uninstalled on next run)
- Format: `brew "package-name"` or `cask "app-name"` or `mas "App Name", id: 123456`

### macOS-Specific
- All scripts are wrapped in `{{ if eq .chezmoi.os "darwin" -}}...{{ end -}}` checks
- The repository is designed for macOS only - scripts won't run on Linux/Windows
- System defaults in `run_once_before_01-setup-macos.sh.tmpl` only run once per machine

## Architecture

### Directory Structure
```
~/.local/share/chezmoi/           # Source directory
├── dot_zshrc                     # → ~/.zshrc
├── dot_config/
│   ├── nvim/lua/                 # Custom Neovim configs (overlay on LazyVim)
│   ├── atuin/config.toml         # Shell history sync config
│   ├── ghostty/config            # Terminal emulator config
│   ├── private_starship.toml     # Encrypted prompt config
│   └── private_zellij/           # Encrypted multiplexer config
└── run_*_before_*.sh.tmpl        # Setup scripts (templated)
```

### Neovim Configuration
- Uses [LazyVim](https://www.lazyvim.org/) distribution as base
- Custom configs in `dot_config/nvim/lua/` overlay on top of LazyVim
- LazyVim is cloned by `run_onchange_before_03-install-lazyvim.sh.tmpl`
- Custom configs are applied after LazyVim installation

### Shell Setup
- Oh-My-Zsh with plugins: `git`, `zsh-syntax-highlighting`, `zsh-autosuggestions`
- Starship prompt (configured in `private_starship.toml`)
- Atuin for shell history sync
- Theme: `robbyrussell` (line 11 in `dot_zshrc`)

## Gotchas

1. **Changes don't apply automatically**: Running `chezmoi edit` or modifying files in the source directory does NOT automatically deploy them. Must run `chezmoi apply`.

2. **Package cleanup is destructive**: Running `chezmoi apply` after modifying the Brewfile will REMOVE packages not in the list. Always review `chezmoi diff` first.

3. **Templates require valid syntax**: `.tmpl` files use Go template syntax. Syntax errors will cause `chezmoi apply` to fail.

4. **Encrypted files**: `private_*` files require chezmoi encryption to be set up. If not configured, these files will be skipped during apply.

5. **Script idempotency**: `run_once_*` scripts only run once per machine. To re-run, must delete chezmoi state or use `run_onchange_*` instead.

6. **Working directory matters**: When in `~/.local/share/chezmoi/`, you're editing source files. Changes here don't affect `~/` until `chezmoi apply` is run.
