# Dev Containers Framework

A flexible Docker-based development environment framework that supports multiple container configurations with different base images.

## Features

- Support for multiple container environments with different base images
- Template-based configuration system
- Environment-specific overrides
- Easy activation/deactivation of containers
- GPU support (NVIDIA)
- SSH key forwarding
- Shared and environment-specific configuration

## Installation

Add this framework as a git submodule to your project:

```bash
git submodule add <framework-repo-url> dev-containers
git submodule update --init --recursive
```

## Project Structure

Your project should have the following structure:

```
your-project/
├── dev-containers/              # This framework (submodule)
│   ├── templates/
│   │   ├── Dockerfile.template
│   │   └── docker-compose.template.yml
│   ├── scripts/
│   │   └── container            # Main CLI tool
│   └── README.md
├── container-config/            # Your project-specific configs
│   ├── environments/
│   │   ├── env1/
│   │   │   ├── config.env
│   │   │   ├── Dockerfile.custom  # Optional: pre-template commands
│   │   │   ├── Dockerfile.post    # Optional: post-template commands
│   │   │   └── compose.override.yml  # Optional
│   │   └── env2/
│   │       └── config.env
│   └── shared/
│       └── config.env           # Shared secrets/config
└── .gitignore
```

## Quick Start

1. **Create your first environment:**

```bash
mkdir -p container-config/environments/my-dev
```

2. **Create a config file** (`container-config/environments/my-dev/config.env`):

```bash
# Base Docker image
BASE_IMG=ubuntu:24.04

# Container identification
CONTAINER_NAME=my-dev-container
VOLUME_NAME=my-dev-vol

# Resource limits
CPU_LIMIT=8.0
MEMORY_LIMIT=16G

# System access
PRIVILEGED=false
```

3. **List available environments:**

```bash
./dev-containers/scripts/container list
```

4. **Activate your environment:**

```bash
./dev-containers/scripts/container activate my-dev
```

## Configuration

### Environment Config (`config.env`)

Each environment requires a `config.env` file with these variables:

- `BASE_IMG`: Base Docker image to use
- `CONTAINER_NAME`: Name for the container
- `VOLUME_NAME`: Name for the persistent volume
- `CPU_LIMIT`: CPU limit (e.g., "8.0")
- `MEMORY_LIMIT`: Memory limit (e.g., "16G")
- `PRIVILEGED`: Whether container needs privileged mode ("true"/"false")

### Custom Dockerfile

To add environment-specific packages, create `Dockerfile.custom` and/or `Dockerfile.post` in your environment directory.

**Option 1: Partial Dockerfile (Recommended - No Duplication)**

Just add your custom Docker commands without the boilerplate. The framework will automatically merge these with the base template:

```dockerfile
# =============================================================================
# Custom commands for this environment
# No need to include FROM, WORKDIR, or base tools - they're added automatically
# =============================================================================

# Install CUDA
RUN apt install -y wget gnupg && \
    wget https://developer.download.nvidia.com/.../cuda-keyring_1.1-1_all.deb && \
    dpkg -i cuda-keyring_1.1-1_all.deb && \
    apt update && \
    apt install -y cuda-toolkit-12-6

# Set environment variables
ENV PATH=/usr/local/cuda/bin:${PATH}

# Install additional packages
RUN apt install -y python3-pip python3-dev
```

**Option 2: Post-Template Commands (`Dockerfile.post`)**

If you need to install project-specific tools that depend on the standard dev environment (cargo, npm, pip, etc.) already being available, create `Dockerfile.post`:

```dockerfile
# Installed after all standard tools (zsh, neovim, cargo, npm, etc.)
RUN cargo install tree-sitter-cli
RUN pip install torch numpy
RUN npm install -g typescript
```

You can use both `Dockerfile.custom` and `Dockerfile.post` together — custom commands run early (before tools), post commands run at the end (after tools and configs).

**Option 3: Complete Dockerfile (Full Control)**

If you need complete control, include a `FROM` statement. The framework will use this file as-is:

```dockerfile
ARG BASE_IMG
FROM ${BASE_IMG}

WORKDIR /root

# Your complete custom setup here
RUN apt update && apt install -y everything-you-need

# ... rest of your dockerfile
```

### Compose Overrides

For advanced Docker Compose configurations (like GPU support), create `compose.override.yml`:

```yaml
services:
  dev-env:
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
    environment:
      - CUSTOM_VAR=value
```

### Shared Configuration

Create `container-config/shared/config.env` for secrets and configs shared across all environments:

```bash
# Secrets (DO NOT COMMIT!)
GITHUB_TOKEN=your_token
NPM_TOKEN=your_token
```

**Important:** Add `container-config/shared/config.env` to `.gitignore`!

## CLI Usage

### List Environments

```bash
./dev-containers/scripts/container list
```

Shows all available environments and their status.

### Activate Environment

```bash
./dev-containers/scripts/container activate <environment>
```

Starts the container and opens a shell. If the container is already running, just opens a shell.

### Execute Command

```bash
./dev-containers/scripts/container exec <environment> [command]
```

Execute a command in a running container. If no command is provided, opens a shell.

### Deactivate Environment

```bash
./dev-containers/scripts/container deactivate <environment>
```

Stops the container (but doesn't remove it).

### Remove Container

```bash
./dev-containers/scripts/container remove <environment>
```

Stops and removes the container completely.

### View Logs

```bash
./dev-containers/scripts/container logs <environment>
./dev-containers/scripts/container logs <environment> -f  # Follow
./dev-containers/scripts/container logs <environment> --tail 100
```

## Default Tools Included

The base template includes:

- **Shell**: zsh with oh-my-zsh
- **Editor**: Neovim (via bob-nvim)
- **Git**: git, lazygit
- **Tools**: tmux, zoxide, stow, fzf, ripgrep
- **Dev Tools**: rust/cargo, nodejs/npm, python/black
- **Formatters**: prettier, prettierd, clang-format

## Customization

### Modifying the Base Template

The base template is in `dev-containers/templates/Dockerfile.template`. You can modify it to change what's included in all your containers, or create environment-specific `Dockerfile.custom` files to add packages to specific environments.

### Custom Dotfiles

The base template clones dotfiles from a configurable git repository. To use your own dotfiles, modify the Dockerfile template or override it in your environment's `Dockerfile.custom`.

## Advanced Usage

### Multiple Containers Running Simultaneously

The framework supports running multiple containers at the same time:

```bash
./dev-containers/scripts/container activate env1
# Open new terminal
./dev-containers/scripts/container activate env2
```

### SSH Key Forwarding

SSH keys are automatically forwarded using ssh-agent. The framework looks for `~/.ssh/id_ed25519` by default.

### Volume Persistence

Each environment has its own named volume for persistent storage. Work is saved in `/root/workspace` inside the container.

## Troubleshooting

### Container won't start

Check your configuration:
```bash
cat container-config/environments/<env>/config.env
```

View container logs:
```bash
./dev-containers/scripts/container logs <env>
```

### GPU not detected

Ensure:
1. NVIDIA Docker runtime is installed
2. `compose.override.yml` includes GPU configuration
3. `PRIVILEGED=true` in config.env

### Permission issues

If you encounter permission issues, ensure the container is running with appropriate privileges.

## Contributing

This framework is designed to be extended. Feel free to submit improvements!
