# Scaffold Instructions: `UGV-local-environment-variables` Repository

This document defines the initial structure and conventions for:

- `https://github.com/PrivateBungo/UGV-local-environment-variables`

The repository is currently empty; use this as the bootstrap scaffold.

## 1) Repository purpose

Store **robot-specific operational environment configuration** and **default fallback configuration**.

This repository should not contain firmware/app source code.

## 2) Proposed directory structure

```text
UGV-local-environment-variables/
├─ README.md
├─ defaults/
│  └─ default.env
├─ robots/
│  ├─ .gitkeep
│  └─ example-robot.env.example
├─ schemas/
│  └─ env-keys.md
└─ scripts/
   └─ validate-env.sh
```

## 3) Minimum required files

### `defaults/default.env`

Must always exist. This is the fallback for new hostnames.

Suggested baseline keys (example):

```env
UGV_PROFILE=default
CAMERA_PRIMARY=/dev/video0
CAMERA_FPS=30
CONTROL_MAX_LINEAR_MPS=0.5
CONTROL_MAX_ANGULAR_RADPS=0.8
FEATURE_AUTONOMY=false
ACL_PROFILE=standard
```

### `robots/<hostname>.env`

One file per robot, keyed by exact hostname. Example:

- `robots/ugv-042.env`

Only override keys that differ from defaults if your host-infra merge logic supports overlays; otherwise include full key sets.

### `schemas/env-keys.md`

Define and document:

- key name
- type
- allowed range/enum
- default
- consuming service

This becomes the contract between infra and application services.

### `scripts/validate-env.sh`

Provide a lightweight validator that:

- checks file naming (`robots/*.env`)
- checks syntax (`KEY=VALUE`)
- rejects duplicate keys per file
- optionally checks required key presence

Use this script in CI before merges.

## 4) Naming and formatting conventions

- Use uppercase snake-case keys.
- Use UTF-8 text, LF line endings.
- Avoid inline secrets in plaintext.
- Keep comments focused and operational.

Example style:

```env
# Camera transport
CAMERA_PRIMARY=/dev/video0
CAMERA_CODEC=h264
```

## 5) Bootstrap workflow for a brand-new robot

1. Robot is provisioned with hostname + deploy key.
2. Host infra attempts `robots/<hostname>.env`.
3. If missing, host infra falls back to `defaults/default.env`.
4. Ops creates `robots/<hostname>.env` in this repo.
5. Next reconcile picks up robot-specific file automatically.

## 6) Suggested initial README content for the env repo

Include:

- repository intent
- directory contract (`defaults/`, `robots/`, `schemas/`, `scripts/`)
- required fallback file (`defaults/default.env`)
- robot file naming rule (`robots/<hostname>.env`)
- note about future encrypted per-robot files

## 7) Future enhancement: per-robot encrypted env files

When ready, extend scaffold with encrypted variants, for example:

- `robots/<hostname>.env.age`

Keep plaintext defaults if needed for bootstrap, or split defaults into non-secret + encrypted secret fragments.

The host infra should decrypt only the target robot file and never pull secrets for other robots into runtime.
