# Quadlet Operations

These commands are for user-level Quadlet units managed through systemd.

## Reload Generated Units

```bash
systemctl --user daemon-reload
```

## Start One Stack

Start in this order:

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

Example for `rsshub`:

```bash
systemctl --user start rsshub-pod.service
systemctl --user start tailscale-rsshub.service
systemctl --user start redis.service
systemctl --user start rsshub.service
```

## Stop One Stack

Stop in reverse order:

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

## Temporary Disable

```bash
systemctl --user mask --runtime <app>.service tailscale-<stack>.service <stack>-pod.service
systemctl --user stop <app>.service tailscale-<stack>.service <stack>-pod.service
```

Restore with:

```bash
systemctl --user unmask <app>.service tailscale-<stack>.service <stack>-pod.service
```

## Check Status

```bash
systemctl --user status <stack>-pod.service tailscale-<stack>.service <app>.service
podman ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
```

## Auto Update

Enable the user timer once:

```bash
systemctl --user enable --now podman-auto-update.timer
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
