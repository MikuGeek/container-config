# Quadlet Layout Reference

This file is a compact reference for the current Quadlet structure used in this repository.

## Canonical Stack Location

- `stacks/<stack>/`

## Canonical Per-Stack Files

- optional `<stack>.target` as the stack-level control entrypoint
- `<stack>.pod`
- `tailscale-<stack>.container` when Tailscale is used
- one or more app `.container` files
- optional `.env.app` symlink for app-specific variables

## Canonical Stack Control Pattern

- put `<stack>.target` in `stacks/<stack>/`
- link the target into `~/.config/systemd/user/`
- link `~/.config/systemd/user/default.target.wants/<stack>.target` to that user target
- keep `.pod` and `.container` source links in `~/.config/containers/systemd/`
- add `PartOf=<stack>.target` to the member `.pod` and `.container` files
- remove member `WantedBy=default.target` when the stack target is the entrypoint

## Canonical Env Locations

- shared infrastructure env: `env/shared/`
- app env: `env/apps/`

## Canonical Symlink Pattern

- `stacks/<stack>/.env.app -> ../../env/apps/<stack>.env`

## Canonical Tailscale Env Reference

Use this in Tailscale sidecars:

`EnvironmentFile=%h/container-config/env/shared/tailscale.env`

## Canonical App Env Reference

Use this in app containers when app-local env is needed:

`EnvironmentFile=%h/container-config/stacks/<stack>/.env.app`

## Canonical Naming Rule

- main app container: `<stack>`
- supporting container: `<stack>-<service>`

## Example

For `rsshub`:

- pod: `rsshub.pod`
- app container: `rsshub.container`
- sidecar: `tailscale-rsshub.container`
- support container: `redis.container`
- runtime container names:
  - `rsshub`
  - `rsshub-tailscale`
  - `rsshub-redis`

Current stacks using stack targets:

- `n8n`
- `microbin`
- `qbittorrent`
- `calibre-web`
