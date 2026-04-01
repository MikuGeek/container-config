# Repository Rules For AI And Automation

This file is the AI-first operating guide for this repository.

## Purpose

- This repository is the source of truth for Podman stack configuration.
- The repository stores config, templates, and env layout conventions.
- Runtime data does not belong in this repository.

## Current Layout

- `stacks/<stack>/`: real stack definitions and Quadlet source files
- `env/shared/`: shared infrastructure env files
- `env/apps/`: centralized per-app env files
- `examples/compose/`: compose reference examples
- `examples/quadlet-stack/`: Quadlet stack template files
- `QUADLET-OPERATIONS.md`: operational commands for user systemd Quadlet units

## Stack Rules

- Edit real stack files only under `~/container-config/stacks/<stack>/`.
- Keep one stack per directory.
- Keep runtime data under `~/container-data/<stack>/`.
- Do not store runtime state in this repo.
- Prefer small, reversible changes.

## Quadlet Rules

- Quadlet source files live in `stacks/<stack>/`.
- Per stack, prefer this shape:
  - optional `<stack>.target` as the stack-level control entrypoint
  - `<stack>.pod`
  - `tailscale-<stack>.container` when Tailscale is used
  - one or more app `.container` files
- User Quadlet links live in `~/.config/containers/systemd/`.
- User stack targets live in `~/.config/systemd/user/`.
- When a stack has `<stack>.target`, make that target the only `default.target` entrypoint instead of individual Quadlet units.
- Do not link plain systemd targets into `~/.config/containers/systemd/`; that directory is only for Quadlet source files.
- Generated systemd units under `/run/user/.../systemd/generator/` must not be edited.

## Container Naming Rules

- Main app container name: `<stack>`
- Supporting container name: `<stack>-<service>`
- Examples:
  - `n8n`
  - `n8n-tailscale`
  - `rsshub`
  - `rsshub-redis`
  - `rsshub-tailscale`

## Env Rules

- Do not commit real secrets.
- Shared infrastructure env belongs in `env/shared/`.
- Per-app env belongs in `env/apps/`.
- Stack directories may expose app env through symlinks only:
  - `stacks/<stack>/.env.app -> ../../env/apps/<stack>.env`
- Tailscale sidecars should read:
  - `%h/container-config/env/shared/tailscale.env`

## Podman Rules

- Podman runs rootless.
- Preserve `:Z,U` or `:z,U` on bind mounts unless there is a clear reason to change them.
- If a base image already supports `PUID=1000` and `PGID=1000`, do not add SELinux relabel flags to external bind mounts for that app without a clear reason.

## Tailscale Rules

- When Tailscale is used for web access, do not publish the app's main HTTP port directly.
- Expose the web service through the Tailscale sidecar serve configuration.
- If a stack needs extra published ports in addition to Tailscale web access, keep those non-web published ports on the pod or the Tailscale-facing side of the stack, not on a separately exposed app container.

## Git Rules

- Commit only intended config changes under `~/container-config`.
- Never commit files from `~/container-data`.
- Never commit real env files, secrets, or generated state.
- Prefer updating docs when layout or workflow changes.

## Validation Rules

- Validate the affected stack after config changes when practical.
- For Quadlet-managed stacks, prefer checking:
  - `systemctl --user status`
  - `podman ps`
  - `journalctl --user -u <unit>`
- For stack targets, validate both `start <stack>.target` and `stop <stack>.target`, then confirm the final state after a short wait instead of reading only the intermediate `deactivating` state.
- For a full host-level stack check, validate all stack targets and confirm both systemd active states and `podman ps` output.
- For compose reference files, use `podman compose config` when validating syntax.

## Auto-Update Rules

- Quadlet containers use `io.containers.autoupdate=registry` when automatic image refresh is desired.
- The user-level `podman-auto-update.timer` is enabled.
- The current local policy is weekly auto-update via a user systemd override.
- If you change the update cadence, also update `QUADLET-OPERATIONS.md` and `LESSONS.md`.

## Migration Note

- The repository used to keep stack directories at the repo root.
- The current canonical layout is `stacks/<stack>/`.
- The repository also used to mix env files into stack directories.
- The current canonical env layout is `env/shared/` and `env/apps/`.
