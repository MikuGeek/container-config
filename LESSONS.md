# Lessons And Writing Standard

This file stores small, practical lessons learned while working in this repository.

## Writing Standard

Use these rules when adding a new lesson:

1. Keep each lesson short and concrete.
2. Record only things that were observed in practice, not guesses.
3. Focus on behavior that can save time during future edits, restarts, migrations, or debugging.
4. Prefer this structure for each lesson:
   - `Title`: one-line summary
   - `Context`: where it happened
   - `Problem`: what failed or behaved unexpectedly
   - `Cause`: why it happened
   - `Fix`: what resolved it
   - `Rule`: the reusable takeaway
5. When the lesson is about a specific file format or tool, include the exact syntax that worked.
6. If the lesson becomes a permanent repo rule, also update `AGENTS.md`, `README.md`, or `QUADLET-OPERATIONS.md` as appropriate.
7. Keep examples minimal and redact real secrets.

## Lessons

### 1. Quote cron expressions in Quadlet `Environment=` lines

- Title: Quote cron expressions with spaces
- Context: `stacks/metapi/metapi.container`
- Problem: `metapi.service` started, then crashed immediately with a `node-cron` parsing error.
- Cause: Lines like `Environment=CHECKIN_CRON=0 2 * * *` were split by systemd/Quadlet, so the application received a broken value.
- Fix: Use quoted values:
  - `Environment="CHECKIN_CRON=0 2 * * *"`
  - `Environment="BALANCE_REFRESH_CRON=0 * * * *"`
- Rule: Any `Environment=` value containing spaces must be quoted in Quadlet files.

### 2. Do not use `systemctl enable` for generated Quadlet services

- Title: Quadlet services are generated, not regular static units
- Context: user systemd + Quadlet
- Problem: `systemctl --user enable ...` failed for Quadlet-generated services.
- Cause: Generated units are transient/generated from `.container` and `.pod` source files.
- Fix: Put `[Install] WantedBy=default.target` in the Quadlet source file and run `systemctl --user daemon-reload`.
- Rule: Manage enable behavior through Quadlet source files, not by enabling generated services directly.

### 3. Start pod-based stacks in order

- Title: Pod, then sidecar, then app
- Context: Quadlet stacks using `.pod` plus a Tailscale sidecar
- Problem: Bulk restarts caused pod replacement races, canceled jobs, and partially started stacks.
- Cause: Pod, sidecar, and app units were restarted together without a stable order.
- Fix: Start stacks in this order:
  1. `<stack>-pod.service`
  2. `tailscale-<stack>.service`
  3. app service such as `n8n.service` or `rsshub.service`
- Rule: For full restarts or recreates, use ordered startup and reverse-order shutdown.

### 4. Keep `ContainerName=` globally unique

- Title: Container names are global in Podman
- Context: multiple Quadlet stacks on one host
- Problem: Generic names such as `redis` can collide with another stack later.
- Cause: Podman container names are global, not scoped to a pod or stack directory.
- Fix: Use this naming rule:
  - main app container: `<stack>`
  - supporting container: `<stack>-<service>`
- Rule: Never use generic supporting names like `redis` without a stack prefix.

### 5. Separate shared env from app env

- Title: Shared infra env and app env should not be mixed
- Context: migration from stack-local `.env` files to centralized env layout
- Problem: Some stack env files mixed shared variables like `TS_AUTHKEY` with app-specific secrets.
- Cause: Earlier compose-oriented layout stored all variables together.
- Fix:
  - shared infra env: `env/shared/tailscale.env`
  - app env: `env/apps/<stack>.env`
  - stack-local symlink: `stacks/<stack>/.env.app -> ../../env/apps/<stack>.env`
- Rule: Shared infrastructure secrets belong in `env/shared/`; app secrets belong in `env/apps/`.

### 6. Do not mix active compose control and active Quadlet control for the same stack

- Title: One active control plane per stack
- Context: `metapi` migration from compose to Quadlet
- Problem: Compose and Quadlet versions of the same stack can conflict in container names, env expectations, and restart behavior.
- Cause: Both systems try to manage the same application lifecycle from different definitions.
- Fix: Stop or remove the compose-managed containers before starting the Quadlet-managed stack.
- Rule: A stack should be actively managed by either compose or Quadlet, not both at the same time.

### 7. Quadlet auto-update needs both labels and the user timer

- Title: Auto-update is not active until the timer is enabled
- Context: Quadlet-managed stacks with `io.containers.autoupdate=registry`
- Problem: Containers had the correct autoupdate label, but no automatic update checks were actually running.
- Cause: The user-level `podman-auto-update.timer` was disabled.
- Fix:
  - Keep `Label=io.containers.autoupdate=registry` in each `.container` file.
  - Enable the user timer with:
    - `systemctl --user enable --now podman-auto-update.timer`
  - Validate with:
    - `podman auto-update --dry-run`
- Rule: For Quadlet auto-update to work, both the container label and the user timer must be present.
