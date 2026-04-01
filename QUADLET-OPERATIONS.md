# Quadlet Operations

These commands are for user-level Quadlet units managed through systemd.

## Reload Generated Units

```bash
systemctl --user daemon-reload
```

If a stack uses `<stack>.target`, make sure the target is linked into `~/.config/systemd/user/`, not `~/.config/containers/systemd/`.

## Start One Stack

Prefer a stack target when one exists:

```bash
systemctl --user start <stack>.target
```

Example for `rsshub`:

```bash
systemctl --user start rsshub.target
```

The `rsshub.target` file itself should be linked into `~/.config/systemd/user/rsshub.target`.

If a stack does not yet have a target, start in this order:

```bash
systemctl --user start <stack>-pod.service
systemctl --user start tailscale-<stack>.service
systemctl --user start <app>.service
```

Example for `n8n`:

```bash
systemctl --user start n8n-pod.service
systemctl --user start tailscale-n8n.service
systemctl --user start n8n.service
```

## Stop One Stack

Prefer a stack target when one exists:

```bash
systemctl --user stop <stack>.target
```

Example for `rsshub`:

```bash
systemctl --user stop rsshub.target
```

For Podman-generated units, a stopped stack may briefly show a member unit as `failed` during teardown even when the containers are already gone. Confirm with `podman ps` if needed.

If a stack does not yet have a target, stop in reverse order:

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

If a stack does not yet have a target:

```bash
systemctl --user mask --runtime <app>.service tailscale-<stack>.service <stack>-pod.service
systemctl --user stop <app>.service tailscale-<stack>.service <stack>-pod.service
```

Restore with:

```bash
systemctl --user unmask <app>.service tailscale-<stack>.service <stack>-pod.service
```

## Check Status

Prefer checking the stack target and its members when one exists:

```bash
systemctl --user status <stack>.target <stack>-pod.service tailscale-<stack>.service <app>.service
podman ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

For `rsshub`:

```bash
systemctl --user status rsshub.target rsshub-pod.service tailscale-rsshub.service redis-rsshub.service rsshub.service
podman ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

If a stack does not yet have a target:

```bash
systemctl --user status <stack>-pod.service tailscale-<stack>.service <app>.service
podman ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

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

For `rsshub`:

```bash
journalctl --user -u rsshub-pod.service -u tailscale-rsshub.service -u redis-rsshub.service -u rsshub.service -n 120 --no-pager
```

## Do Not Do This

- Do not bulk restart many stacks at once.
- Do not mix compose control and Quadlet control for the same active stack.
- Do not restart pod, sidecar, and app together in one command unless you are intentionally recreating the stack.
- Do not edit generated `.service` files under `/run/user/.../systemd/generator/`.
