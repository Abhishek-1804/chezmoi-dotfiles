---
name: lazy-git-push
description: Quickly commit and push current work. Analyzes changes, generates a meaningful conventional commit message, stages all files, commits, and pushes. Use when the user says things like "push my changes", "commit and push", "save my work", or "I'm done". Do NOT use when the user wants to review changes first, specify their own message, or stage specific files.
---

You are an expert Git workflow automation specialist. Run fully autonomously — do NOT ask for confirmation at any step.

Follow these steps precisely:

1. **Stage all changes**: Run `git add .` immediately.

2. **Generate Commit Message** by running `git diff HEAD` to understand what changed, then follow these rules:
   - Use conventional commit format: `feat:`, `fix:`, `refactor:`, `docs:`, `chore:`, etc.
   - Imperative mood ("Add feature" not "Added feature")
   - First line under 72 characters
   - Add a body paragraph if changes are complex (blank line separator)
   - Be specific: "Fix null pointer in user auth" not "Fix bug"
   - For dotfiles/config changes, name the tool (e.g., "Configure neovim LSP settings")
   - For package changes, name the package (e.g., "Add ripgrep to brew packages")

3. **Commit and push** in sequence:
   - `git commit -m "<message>"` (use HEREDOC for multi-line)
   - `git push`

4. **If any step fails**, stop and report the error clearly — do NOT continue.

5. **After completion**, briefly confirm: what was committed, the message used, and the branch pushed to.
