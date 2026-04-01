# Container Configuration

This repository is the source of truth for local Podman stack configuration.

It is organized to be easy to manage for both humans and automation:

- real stacks live under `stacks/`
- real env files are centralized under `env/`
- examples and templates live under `examples/`
- user Quadlet operations are documented separately

This `README.md` is the human-readable English overview.
For AI and automation rules, see `AGENTS.md`.

## Repository Layout

- `stacks/<stack>/`
  - real stack configuration
  - Quadlet source files
  - stack-local support files such as `tailscale-serve.json`
  - app env symlink when needed
- `env/shared/`
  - shared infrastructure env files
  - current example: `tailscale.env`
- `env/apps/`
  - centralized app env files
  - one real app env file per stack when needed
- `examples/compose/`
  - compose reference examples
- `examples/quadlet-stack/`
  - reusable Quadlet template files for a new stack
- `QUADLET-OPERATIONS.md`
  - safe operational commands for user systemd Quadlet units

## Current Conventions

### Stack files

Each real stack should live in `stacks/<stack>/`.

Typical files:

- `<stack>.target` when the stack is managed through one user systemd entrypoint
- `<stack>.pod`
- `tailscale-<stack>.container` when the stack uses Tailscale
- one or more app `.container` files
- `.env.app` symlink when the app needs stack-specific env

### Env files

Shared infrastructure env files live in `env/shared/`.

App env files live in `env/apps/` and are exposed inside stack directories through symlinks such as:

- `stacks/rsshub/.env.app -> ../../env/apps/rsshub.env`
- `stacks/metapi/.env.app -> ../../env/apps/metapi.env`

### Container names

The naming rule is:

- main app container: `<stack>`
- supporting container: `<stack>-<service>`

Examples:

- `n8n`
- `n8n-tailscale`
- `rsshub`
- `rsshub-redis`
- `rsshub-tailscale`

## Git-Friendly Rules

- Do not commit real secrets.
- Do not commit runtime data.
- Keep real env files ignored by Git.
- Keep symlink entry points versioned when useful.
- Prefer small, readable config changes.

## Runtime Data

Runtime data belongs under:

- `~/container-data/<stack>/`

It does not belong in this repository.

## User Quadlet Links

Quadlet source files in this repo are linked into:

- `~/.config/containers/systemd/`

Stack-level user systemd targets are linked into:

- `~/.config/systemd/user/`

Generated units are then managed through user systemd.

When a stack has a `<stack>.target`, prefer operating and enabling that target instead of enabling each generated unit separately.

At the moment this repository uses stack targets for all active stacks:

- target-managed: `n8n`, `microbin`, `qbittorrent`, `calibre-web`, `rsshub`, `metapi`

## Auto Update

Quadlet containers that should auto-update use the label:

- `io.containers.autoupdate=registry`

The repository currently assumes a user-level Podman auto-update timer with a weekly schedule.
The operational commands are documented in `QUADLET-OPERATIONS.md`.

## Further Reading

- `AGENTS.md`: AI and automation rules
- `QUADLET-OPERATIONS.md`: safe commands for start, stop, restart, and temporary disable
- `env/README.md`: env layout details
- `examples/quadlet-stack/README.md`: template usage
