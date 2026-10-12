# Repository Instructions

## Cross-Platform Contract

This repository is shared by separate Windows and Arch Linux checkouts. Keep one
shared source branch unless the user explicitly requests a platform fork.

- Detect the current OS and available executables before issuing installation,
  package-management, GUI, path, or shell commands.
- Never commit machine-specific absolute paths, home directories, credentials,
  environment files, caches, logs, dependency directories, or build output.
- Keep runtime configuration machine-local. In particular, do not copy a
  Windows `~/.codex/config.toml` MCP command path into Linux, or vice versa.
- Install dependencies and build native or generated artifacts independently on
  each OS. Never share `node_modules`, virtual environments, or compiled output.
- Prefer portable source and runtime platform detection. When platform-specific
  behavior is necessary, keep Windows and Linux implementations together using
  explicit `.ps1` and `.sh` entry points or well-scoped conditional logic.
- Use repository-relative paths in tracked files. In documentation and example
  configuration, use placeholders or environment variables instead of usernames.
- Preserve UTF-8. In Windows PowerShell, specify UTF-8 explicitly when a command's
  default encoding is ambiguous.
- Use path names valid on both systems: no Windows reserved names, trailing dots
  or spaces, invalid Windows characters, or paths that differ only by case.
- Preserve executable bits for shell scripts and use `#!/usr/bin/env bash` when
  Bash is required. Do not assume that an interactive zsh session runs scripts.

## Host-Specific Guidance

- Windows automation uses PowerShell. Prefer executable shims such as `npm.cmd`
  and `npx.cmd` when the PowerShell execution policy blocks `.ps1` shims.
- Linux is Arch Linux x86_64 with zsh, KDE Plasma on Wayland, pacman, and Nix.
  Prefer project-local tooling first, then pacman for system packages. Use Nix
  when the project already defines a Nix environment or the user requests it;
  do not manage the same dependency with pacman and Nix without a clear reason.
- Do not assume X11 or `xdotool` on Linux. Detect the display session and prefer
  Wayland-compatible automation or desktop portals.
- Do not assume NVIDIA or CUDA. Detect acceleration hardware and drivers before
  selecting GPU-specific dependencies.
- The WPS MCP server uses PowerShell COM on Windows and add-on polling on Linux.
  Keep the shared submodule revision identical, but install its dependencies,
  build it, install its WPS integration, and register its MCP path per OS.

## Submodules

- Treat submodule revisions as shared source state, not machine configuration.
- Do not commit ignored build output or local configuration inside a submodule.
- Update a submodule pointer deliberately and verify the parent repository diff.

