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

### 8. `podman auto-update --dry-run` can look idle while checking registries

- Title: Dry-run may appear stuck before the final table
- Context: manual validation of Podman auto-update
- Problem: `podman auto-update --dry-run` seemed to have no output for a while.
- Cause: Podman was still querying multiple remote registries and only printed the summary table after those checks completed.
- Fix: Wait for the final table output or use debug logging to confirm registry checks are still running.
- Rule: Treat `podman auto-update --dry-run` as a network-bound check; lack of immediate output does not mean it is broken.

### 9. Stack targets must be linked through user systemd, not Quadlet source links

- Title: Put `<stack>.target` in `~/.config/systemd/user/`
- Context: stack target migration for `n8n`, `microbin`, `qbittorrent`, and `calibre-web`
- Problem: Plain systemd targets are not loaded when linked into `~/.config/containers/systemd/`.
- Cause: That directory is consumed by the Podman Quadlet generator for `.pod` and `.container` sources, not normal systemd target units.
- Fix:
  - Keep source files in `stacks/<stack>/<stack>.target`
  - Link them into `~/.config/systemd/user/<stack>.target`
  - Link `default.target.wants/<stack>.target` to the user target file
- Rule: Install stack-level targets through user systemd, not the Quadlet source directory.

### 10. A stack target can replace direct `WantedBy=default.target` on members

- Title: Use one target as the default stack entrypoint
- Context: target migration for `n8n`, `microbin`, `qbittorrent`, and `calibre-web`
- Problem: Managing pod, sidecar, and app units individually was repetitive and made the default startup path harder to understand.
- Cause: Each member unit was installed directly into `default.target`.
- Fix:
  - Add `<stack>.target` with `Wants=` for the stack members
  - Add `PartOf=<stack>.target` to the member `.pod` and `.container` files
  - Remove member `WantedBy=default.target`
  - Enable only the stack target for default startup
- Rule: For stable stacks with one pod, one sidecar, and one app, prefer a single stack target as the default entrypoint.

### 11. Target stop validation needs a delayed recheck

- Title: Read the settled state, not the transition state
- Context: validating `stop <stack>.target` on `n8n`, `microbin`, `qbittorrent`, and `calibre-web`
- Problem: Immediate checks after `systemctl --user stop <stack>.target` showed `deactivating` units and still-running containers.
- Cause: Podman-generated pod and container units need a short amount of time to stop and remove the pod infra container.
- Fix:
  - Wait a few seconds after the stop command
  - Re-run `systemctl --user is-active ...`
  - Re-run `podman ps`
- Rule: Treat the first post-stop read as an intermediate state; confirm the settled state before deciding a target stop failed.

### 12. Full-stack checks must include both target-managed and non-target stacks

- Title: Check all control entrypoints on mixed hosts
- Context: host-level startup verification after migrating most stacks to stack targets
- Problem: Looking only at target units missed `metapi`, which still uses direct `.pod` and `.container` control.
- Cause: The repository now has a mixed control model: most stacks use `<stack>.target`, but `metapi` still uses direct Quadlet units.
- Fix:
  - Check target-managed stacks via their `.target` units
  - Check `metapi` via `metapi-pod.service`, `tailscale-metapi.service`, and `metapi.service`
  - Confirm with `podman ps`
- Rule: When validating the whole host, include every active control entrypoint, not just stack targets.

### 13. Some startup logs are warnings, not blockers

- Title: Distinguish startup warnings from real failures
- Context: full-stack start of `n8n`, `calibre-web`, and Tailscale sidecars
- Problem: Service logs showed warnings that looked alarming during startup.
- Cause:
  - `n8n` warns about deprecated env and missing Python internal runner support
  - `calibre-web` downloads and installs its Docker mod on startup
  - Tailscale sidecars emit periodic network probe warnings
- Fix:
  - Confirm the unit remains `active`
  - Confirm the container remains `Up` in `podman ps`
  - Treat these logs as non-blocking unless the service actually exits or loops
- Rule: Do not treat every startup warning as a failure; correlate logs with final unit and container state.
