# Quadlet Operations

These commands are for user-level Quadlet units managed through systemd.

## Reload Generated Units

```bash
systemctl --user daemon-reload
```

If a stack has `<stack>.target`, make sure the target is linked into `~/.config/systemd/user/` and `~/.config/systemd/user/default.target.wants/`.

## Start One Stack

Prefer a stack target when one exists:

```bash
systemctl --user start <stack>.target
```

Current target-managed stacks in this repository:

- `n8n`
- `microbin`
- `qbittorrent`
- `calibre-web`
- `rsshub`
- `metapi`

Examples:

```bash
systemctl --user start n8n.target
systemctl --user start microbin.target
```

If you are working on an older stack that does not yet have a target, start in this order:

```bash
systemctl --user start <stack>-pod.service
systemctl --user start tailscale-<stack>.service
systemctl --user start <app>.service
```

Legacy example:

```bash
systemctl --user start n8n-pod.service
systemctl --user start tailscale-n8n.service
systemctl --user start n8n.service
```

Example for `metapi`:

```bash
systemctl --user start metapi.target
```

## Stop One Stack

Prefer a stack target when one exists:

```bash
systemctl --user stop <stack>.target
```

After stopping a target, wait a few seconds before checking status so Podman-generated units can finish pod removal.

If you are working on an older stack that does not yet have a target, stop in reverse order:

```bash
systemctl --user stop <app>.service
systemctl --user stop tailscale-<stack>.service
systemctl --user stop <stack>-pod.service
```

## Restart Only The App

```bash
systemctl --user restart <app>.service
```

Use this when only app configuration changed.

## Recreate One Stack Safely

Use this order when you intentionally want a full recreate:

```bash
systemctl --user stop <app>.service
systemctl --user stop tailscale-<stack>.service
systemctl --user stop <stack>-pod.service
podman rm -f <stack> <stack>-tailscale <stack>-infra
podman pod rm -f <stack>
systemctl --user daemon-reload
systemctl --user start <stack>-pod.service
systemctl --user start tailscale-<stack>.service
systemctl --user start <app>.service
```

Adjust support container names where needed, for example `rsshub-redis`.

For `rsshub`, the support service name is `redis-rsshub.service` and the runtime container name remains `rsshub-redis`.

## Temporary Disable

Prefer masking the stack target when one exists:

```bash
systemctl --user mask --runtime <stack>.target
systemctl --user stop <stack>.target
```

Restore with:

```bash
systemctl --user unmask <stack>.target
```

If you are working on an older stack that does not yet have a target:

```bash
systemctl --user mask --runtime <app>.service tailscale-<stack>.service <stack>-pod.service
systemctl --user stop <app>.service tailscale-<stack>.service <stack>-pod.service
```

Restore with:

```bash
systemctl --user unmask <app>.service tailscale-<stack>.service <stack>-pod.service
```

## Check Status

For target-managed stacks:

```bash
systemctl --user status <stack>.target <stack>-pod.service tailscale-<stack>.service <app>.service
podman ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

If a stack has just been stopped, re-run the command after a short wait to confirm the final inactive state.

Legacy non-target pattern:

```bash
systemctl --user status <stack>-pod.service tailscale-<stack>.service <app>.service
podman ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

## Check All Stacks

Use this when you want a host-level view after startup or maintenance:

```bash
systemctl --user status n8n.target microbin.target qbittorrent.target calibre-web.target rsshub.target metapi.target
podman ps --format 'table {{.Names}}\t{{.Status}}\t{{.Image}}'
```

During startup, some logs are expected warnings rather than failures:

- `n8n` may warn about deprecated env or missing Python internal runner support
- `calibre-web` may spend time downloading and installing its Docker mod
- Tailscale sidecars may emit periodic network probe warnings

Use final unit state and `podman ps` to decide whether the stack is healthy.

## Auto Update

Enable the user timer once:

```bash
systemctl --user enable --now podman-auto-update.timer
```

The current local policy is weekly auto-update through a user override.

Check the effective timer definition:

```bash
systemctl --user cat podman-auto-update.timer
```

Check the next scheduled run:

```bash
systemctl --user list-timers --all | rg 'podman-auto-update|NEXT|LAST'
```

Preview what Podman would update:

```bash
podman auto-update --dry-run
```

Run one update check immediately:

```bash
systemctl --user start podman-auto-update.service
```

## Logs

```bash
journalctl --user -u <stack>-pod.service -u tailscale-<stack>.service -u <app>.service -n 120 --no-pager
```

## Do Not Do This

- Do not bulk restart many stacks at once.
- Do not mix compose control and Quadlet control for the same active stack.
- Do not restart pod, sidecar, and app together in one command unless you are intentionally recreating the stack.
- Do not edit generated `.service` files under `/run/user/.../systemd/generator/`.
