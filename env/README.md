# Environment Layout

This repository uses a split env layout so that:

- humans can edit env values by project
- automation can manage env values centrally
- Git can track the structure without tracking real secrets

## Shared Env Files

- `env/shared/`: shared infrastructure env files

Current shared file:

- `env/shared/tailscale.env`

Example:

```env
TS_AUTHKEY=tskey-xxxxxxxxxxxxxxxx
```

## App Env Files

- `env/apps/`: centralized real per-app env files

Examples:

- `env/apps/rsshub.env`
- `env/apps/metapi.env`

## Per-Stack Entry Points

Each stack can expose its app env through a symlink inside the stack directory:

- `stacks/rsshub/.env.app -> ../../env/apps/rsshub.env`
- `stacks/metapi/.env.app -> ../../env/apps/metapi.env`

This keeps project-local editing convenient while keeping the real env files centralized.

## Git Behavior

- Real env files are ignored by Git.
- Example env files can be versioned.
- Symlink entry points such as `.env.app` can be versioned when useful.

## Rule Of Thumb

- Shared infra secret: `env/shared/`
- App-specific secret: `env/apps/`
- Stack-local reference: `.env.app` symlink
