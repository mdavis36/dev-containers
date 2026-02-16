# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A Docker-based development environment framework designed to be added as a **git submodule** to other projects. It provides a CLI tool to manage multiple isolated dev containers with different base images, shared tooling, and per-environment customization.

## Repository Structure

- `scripts/container` — Python 3 CLI tool (single file, ~400 lines). The `ContainerManager` class handles all container lifecycle operations.
- `templates/Dockerfile.template` — Base Dockerfile (Ubuntu 24.04, clang-20, zsh, neovim via bob-nvim, rust/cargo, node/npm, lazygit, etc.)
- `templates/docker-compose.template.yml` — Compose template with variable substitution (`${VAR}` and `${VAR:-default}` syntax)

This repo has **no build system, tests, linting, or CI**. The only dependency beyond Python stdlib is PyYAML.

## CLI Commands

Run from the **parent project** (not from this repo):

```bash
./dev-containers/scripts/container list
./dev-containers/scripts/container activate <env>
./dev-containers/scripts/container deactivate <env>
./dev-containers/scripts/container exec <env> [command]
./dev-containers/scripts/container remove <env>
./dev-containers/scripts/container logs <env> [-f] [--tail N]
```

## Architecture

**Configuration resolution order:** shared config (`container-config/shared/config.env`) is loaded first, then environment-specific config (`container-config/environments/<env>/config.env`) overrides it.

**Dockerfile merging:** If `Dockerfile.custom` in an environment directory has no `FROM` statement, it's treated as a partial — its contents are inserted into the base template at the `CUSTOM COMMANDS INSERTION POINT` marker (after `apt update`, before SSH/tools setup). If it has a `FROM`, it's used as a complete replacement. Additionally, `Dockerfile.post` can be used to append commands after the entire template (after tools, configs, and plugins are set up). Both `Dockerfile.custom` and `Dockerfile.post` can be used together.

**Compose override merging:** `compose.override.yml` files are deep-merged into the generated compose config using `_deep_merge()`.

**Generated files** are written to the parent project root as `.docker-compose.<env>.yml` and `.Dockerfile.<env>`.

**Volume layout:** The compose template mounts the parent project at `/root/host`, a named volume at `/root/workspace`, and a Claude projects volume at `/root/.claude/projects`. SSH agent socket is forwarded for git operations.
