# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a LinuxServer.io Docker container for code-server (VS Code running on a remote server, accessible through the browser). The repository contains multiple Dockerfiles and follows the LinuxServer.io build pipeline conventions.

## Build Commands

### Build Standard Image (AMD64)
```bash
docker build --no-cache --pull -t lscr.io/linuxserver/code-server:latest .
```

### Build Custom Image with Claude Code + GCloud
```bash
docker build --no-cache --pull -f Dockerfile.custom -t lscr.io/linuxserver/code-server:latest .
```

### Build ARM64 Variant
First enable QEMU for cross-platform builds:
```bash
docker run --rm --privileged lscr.io/linuxserver/qemu-static --reset
```

Then build with the ARM64 Dockerfile:
```bash
docker build --no-cache --pull -f Dockerfile.aarch64 -t lscr.io/linuxserver/code-server:latest .
```

## Architecture and File Structure

### Key Files and Their Purpose

- **Dockerfile** - Standard AMD64 build (auto-generated, do NOT edit directly)
- **Dockerfile.aarch64** - ARM64 build (auto-generated, do NOT edit directly)
- **Dockerfile.custom** - Custom variant with Claude Code, Node.js, and Google Cloud CLI pre-installed
- **README.md** - Auto-generated from `readme-vars.yml` (do NOT edit directly)
- **Jenkinsfile** - Auto-generated from `jenkins-vars.yml` (do NOT edit directly)
- **readme-vars.yml** - Template variables for generating README.md
- **jenkins-vars.yml** - Build pipeline configuration

### Runtime Configuration Files

The `/root` directory contains files copied into the container at build time:

- **root/etc/s6-overlay/s6-rc.d/init-code-server/run** - Initialization script that sets up sudo access, SSH permissions, and directory structure
- **root/etc/s6-overlay/s6-rc.d/svc-code-server/run** - Service script that launches code-server with appropriate flags
- **root/usr/local/bin/install-extension** - Helper script for installing VS Code extensions (used by Docker mods)

### Custom Dockerfile Features

The `Dockerfile.custom` variant includes:
- **Claude Code**: Installed globally via npm during build
- **Google Cloud CLI**: Installed from official apt repository with GPG keyring
- **Node.js 22.14.0**: Custom Node.js installation supporting both amd64 and arm64 architectures
- **Vertex AI Environment Variables**: Pre-configured for Anthropic's Vertex AI integration
  - `CLAUDE_CODE_USE_VERTEX=1`
  - `CLOUD_ML_REGION=us-east5`
  - `ANTHROPIC_VERTEX_PROJECT_ID=oa-data-btdpexploration-np`
  - `DISABLE_PROMPT_CACHING=0`

## Making Changes

### Modifying the README

NEVER edit README.md directly. Instead:
1. Edit `readme-vars.yml`
2. Use the [Jenkins Builder](https://github.com/linuxserver/docker-jenkins-builder) to regenerate files

Common variables:
- `project_blurb` - Description above the project logo
- `app_setup_block` - "Application Setup" section content
- `param_env_vars` / `opt_param_env_vars` - Environment variable documentation
- `changelogs` - Version history

### Modifying Dockerfiles

When adding packages to ANY Dockerfile:
- Add packages to ALL architecture Dockerfiles (Dockerfile, Dockerfile.aarch64)
- Keep packages in **alphabetical order**
- Update the changelog in `readme-vars.yml`:
  ```yml
  - {date: "DD.MM.YY:", desc: "Added some love to templates"}
  ```

### Modifying Startup Scripts

Changes to files in `/root` (init scripts, service scripts) also require updating the changelog in `readme-vars.yml`.

## Container Runtime Architecture

The container uses **s6-overlay** for process supervision. The initialization flow:

1. **init-code-server** runs first:
   - Creates `/config/{extensions,data,workspace,.ssh}` directories
   - Sets up sudo access if `SUDO_PASSWORD` or `SUDO_PASSWORD_HASH` is set
   - Manages ownership/permissions (skips `/config/workspace` contents for performance)
   - Sets SSH directory permissions (700 for directories, 600 for private keys, 644 for public keys)

2. **svc-code-server** starts the service:
   - Configures authentication (password, hashed password, or none)
   - Sets up proxy domain if specified
   - Launches code-server bound to `0.0.0.0:8443` (or `[::]:8443` for non-root)
   - Opens default workspace (`DEFAULT_WORKSPACE` or `/config/workspace`)

## Environment Variables

Key variables supported by the container:
- `PUID/PGID` - User/group IDs for volume permissions
- `PASSWORD` / `HASHED_PASSWORD` - Web UI authentication
- `SUDO_PASSWORD` / `SUDO_PASSWORD_HASH` - Terminal sudo access
- `PROXY_DOMAIN` - Subdomain proxying configuration
- `DEFAULT_WORKSPACE` - Directory opened by default
- `PWA_APPNAME` - Progressive Web App name

## Testing Changes

Always test builds locally before submitting PRs. Each commit after PR creation triggers a full build pipeline.

## Common Build Issues

### Multi-Architecture Builds

The workflow builds for both `linux/amd64` and `linux/arm64`. Ensure:
- Architecture detection uses `uname -m` correctly
- Both `x86_64` → `amd64`/`x64` and `aarch64` → `arm64` mappings are defined
- Download URLs support both architectures

### User Permissions at Build Time

The `abc` user is created by the LinuxServer.io baseimage **during container startup**, not at build time. Therefore:
- Never use `chown abc:abc` in Dockerfile RUN commands
- Never use `su abc` in Dockerfile RUN commands
- Install packages globally as root, not as a specific user
- User-specific configuration should be done in init scripts (in `/root/etc/s6-overlay/`)

### Volume Mount Points

`/config` is a volume mount point. Do not install application data there during build:
- Anything installed to `/config` at build time will be overwritten at runtime
- Install global packages to system paths (`/usr/local`, `/app`, etc.)
- User data belongs in init scripts that run after volumes are mounted
