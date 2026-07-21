---
name: incus-spawn-dev
description: >
  Working inside a tpl-incus-spawn VM — a pre-configured recursive
  development environment for incus-spawn (isx). Covers what's already
  set up, how to build/test isx, and how nested Incus works.
---

# Incus-Spawn Development Environment

This skill applies when you are working inside a `tpl-incus-spawn` VM
(check: `$ISX_TEMPLATE` is `tpl-incus-spawn`, or `~/.local/bin/isx`
exists and `incus storage list` shows a `cow` pool).

## What's already set up

Everything is pre-configured — do NOT re-install or re-initialize:

- **isx** is built and installed at `~/.local/bin/isx`
- **Nested Incus** is running with a `cow` btrfs storage pool (10GiB)
- **Inner MITM proxy** is running (`isx proxy install` was done)
- **Placeholder credentials** are in `~/.config/incus-spawn/config.yaml`
  (Anthropic API key, GitHub token) — the outer proxy injects real
  credentials on the wire
- **Search paths** include `~/incus-spawn-templates`
- **firewalld** is running for container networking
- **incus-admin group** is active (via sudo/PAM on login)

## Repos available

- `~/incus-spawn` — main isx source (Java/Quarkus/picocli)
- `~/incus-spawn-templates` — template and tool definitions
- `~/incus-spawn-images` — image build infrastructure

## Building and testing isx

```bash
mvnd install -DskipTests        # Fast build (Maven Daemon)
mvnd test                       # Unit tests (no Incus required)
mvnd verify -DskipITs=false     # Integration tests (needs running Incus)
env -u JAVA_TOOL_OPTIONS ~/incus-spawn/install.sh  # Reinstall after changes
```

Always unset `JAVA_TOOL_OPTIONS` when running `install.sh` — the
truststore prepend interferes with Java version detection.

## Working with nested Incus

You can build templates and create instances inside this VM:

```bash
isx build tpl-minimal           # Build a template
isx branch --from tpl-minimal mytest  # Create an instance
isx destroy mytest              # Clean up
```

Templates that require credentials (claude, gh tools) work because
placeholder credentials pass the credential check, and the proxy
chain (inner → outer) handles real credential injection.

## Key constraints

- **HTTPS only for GitHub** — git SSH to github.com doesn't work;
  use HTTPS remotes (the proxy handles authentication via `GH_TOKEN`)
- **VM boot time** — the Incus daemon takes time to start after a
  reboot; if you see "failed to become ready" errors, retry
- **Proxy chain** — HTTPS traffic goes: inner container → inner proxy
  → outer proxy → real server. The inner proxy holds placeholder
  credentials; the outer proxy injects real ones
