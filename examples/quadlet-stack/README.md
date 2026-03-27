# Quadlet Stack Template

Use this template when creating a new Podman Quadlet stack in this repository.

## Files

- `stack.pod.example`: pod definition for the stack
- `tailscale.container.example`: optional Tailscale sidecar
- `app.container.example`: main application container
- `env.app.example`: optional app-specific env example

## Shared Tailscale Env

All Tailscale sidecars in this repository read:

`%h/container-config/env/shared/tailscale.env`

## Recommended Workflow

1. Copy the example files into `stacks/<stack>/`.
2. Rename placeholders like `<stack>` and `<image>`.
3. Create `env/apps/<stack>.env` if the app needs local variables.
4. Expose it inside the stack as `.env.app`.
5. Update volumes, environment variables, ports, and labels.
6. Link the resulting `.pod` and `.container` files into `~/.config/containers/systemd/`.
7. Run `systemctl --user daemon-reload`.
