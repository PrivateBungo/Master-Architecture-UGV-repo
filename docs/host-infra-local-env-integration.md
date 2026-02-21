# Host Infra Instructions: Local Environment Repository Integration

This document is for the `Host-infra-for-UGV-controller` repository.

Goal: integrate robot-local environment configuration retrieval using seeded hostname + deploy key, with deterministic fallback to defaults.

## 1) Inputs expected at bootstrap

Provision each new robot with:

- `ROBOT_HOSTNAME` (or read from `hostnamectl --static`)
- Deploy key that has read access to:
  - Host infra repo
  - `UGV-local-environment-variables` repo
- Optional explicit robot ID (e.g., `/etc/ugv-robot-id`)

Recommended file paths:

- SSH key: `/etc/ugv/keys/deploy_ed25519`
- Known hosts: `/etc/ugv/ssh/known_hosts`
- Runtime env output: `/etc/ugv/runtime/local.env`

## 2) Repository checkout strategy

Implement robot-local pull in the same reconciliation flow used for host infra:

1. Ensure a checkout path exists, for example `/opt/ugv/config/local-env-repo`.
2. Clone if absent, otherwise fetch/reset to pinned branch/ref.
3. Use strict host key verification.
4. Ensure permissions are root-only for key material.

Example non-interactive Git environment:

```bash
export GIT_SSH_COMMAND="ssh -i /etc/ugv/keys/deploy_ed25519 -o IdentitiesOnly=yes -o StrictHostKeyChecking=yes -o UserKnownHostsFile=/etc/ugv/ssh/known_hosts"
```

## 3) Environment file selection contract

Inside the local-environment repo, resolve files in this order:

1. `robots/${ROBOT_HOSTNAME}.env`
2. `defaults/default.env`

Rules:

- If robot-specific file exists, use it.
- If not, fallback to `defaults/default.env`.
- If neither exists, fail convergence with explicit error.

Emit deterministic metadata for observability:

- selected source (`robot` or `default`)
- selected file path
- selected commit SHA

## 4) Runtime materialization

After selecting source env file:

1. Validate syntax (line-oriented `KEY=VALUE`, optional comments).
2. Copy/render to `/etc/ugv/runtime/local.env`.
3. Set ownership/perms (`root:root`, `0640` or stricter).
4. Restart/reload only services that depend on changed keys.

Also persist a state stamp, for example:

- `/var/lib/ugv/local-env.last_sha`
- `/var/lib/ugv/local-env.last_source`

## 5) Systemd integration recommendation

Add a dedicated oneshot unit invoked before compose stack startup:

- `ugv-local-env-sync.service`

Suggested ordering:

- `After=network-online.target tailscaled.service`
- `Before=docker-compose@ugv-stack.service` (or equivalent)

And a timer/reconcile path matching existing host GitOps cadence.

## 6) Failure modes and expected behavior

- **GitHub temporarily unreachable:** keep last-known-good `/etc/ugv/runtime/local.env`; do not block safe local operation.
- **Robot env file missing:** fallback to default and raise a warning event.
- **Default env file missing:** fail converge (configuration contract broken).
- **Invalid env syntax:** reject new file, retain last-known-good, emit alert.

## 7) Security baseline

- Use read-only deploy keys scoped minimally.
- Avoid writing secrets into the local-env repository unless encrypted workflows are introduced.
- Maintain strict file permissions around SSH keys and rendered runtime env files.

## 8) Forward-compatible encryption plan (future)

Design extension point for per-robot encrypted files, for example:

- `robots/${ROBOT_HOSTNAME}.env.age`

Future decrypt flow can use robot-provisioned key material before rendering `/etc/ugv/runtime/local.env`.

Keep current file-selection + fallback contract unchanged so the migration is additive.
