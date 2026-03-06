# Host Infra Instructions: Local-Env Ansible Vault Integration

This document is for the `Host-infra-for-UGV-controller` repository and is written as a **Codex-ready implementation brief**.

Primary goal: during bootstrap, prompt the operator for an Ansible Vault passphrase, validate it immediately, and use decrypted Docker Hub credentials to support authenticated image pulls.

---

## 1) Contract summary (what host-infra must assume)

The local-environment repository exposes:

- non-secret env files under `defaults/default/*.env` (plus optional `<ROBOT_ID>/env/*.env`), and
- one encrypted vault file per selected env directory:
  - `defaults/default/secrets.vault.yml`, or
  - `<ROBOT_ID>/env/secrets.vault.yml`

The encrypted file must decrypt into a predictable YAML schema (defined in `docs/local-env-repo-scaffold.md`).

Host-infra must never assume ad-hoc keys or file names for secrets.

---

## 2) Bootstrap CLI behavior (required)

During `bootstrap/install.sh` (or a helper it calls), implement this sequence:

1. Resolve robot ID (`--robot-id`) and selected env-config directory.
2. Validate SSH access to the local-env repo using the already provisioned deploy key.
3. Clone/pull local-env repo revision to local checkout.
4. Prompt the operator in CLI for vault passphrase (silent input):
   - `read -r -s -p "Enter Ansible Vault passphrase for local env secrets: " VAULT_PASSPHRASE`
5. Write passphrase to a root-only temporary file (`0600`) in `/run/ugv/` or `/tmp/`.
6. Attempt decryption of selected `secrets.vault.yml` via `ansible-vault view`.
7. If decrypt fails, print a clear error and reprompt (bounded retries, e.g., 3).
8. If decrypt succeeds, continue bootstrap and remove temp passphrase file.

Important: do not echo passphrase, do not persist it in git, and do not leave it in shell history.

---

## 3) Validation workflow (required test at bootstrap time)

The bootstrap validation must prove both dependencies:

- Git access works with deploy key.
- Vault passphrase decrypts the selected secrets file.

Recommended checks:

1. `git ls-remote <LOCAL_ENV_REPO_SSH>` with current deploy-key `GIT_SSH_COMMAND`.
2. Verify selected vault file exists in local checkout.
3. `ansible-vault view <selected>/secrets.vault.yml --vault-password-file <tempfile> >/dev/null`

If any step fails, bootstrap must stop with actionable messaging.

---

## 4) Decrypted schema consumption (Docker Hub auth)

Host-infra should parse decrypted YAML and extract Docker Hub credentials from:

- `registry_auth.dockerhub.username`
- `registry_auth.dockerhub.token`
- optional `registry_auth.dockerhub.registry` (default `docker.io`)

Then perform login before compose pull/update:

```bash
docker login "${DOCKERHUB_REGISTRY:-docker.io}" \
  -u "${DOCKERHUB_USERNAME}" \
  --password-stdin <<<"${DOCKERHUB_TOKEN}"
```

Security behavior:

- Keep decrypted content in memory when possible.
- If temporary files are unavoidable, place under root-only runtime path and shred/delete after use.
- Never write token into `.env` files committed to repo.

---

## 5) Runtime/reconcile behavior

For `scripts/ugv-reconcile.sh` + Ansible converge:

- Reuse the same selected env-config path semantics already used for `general.env`.
- If registry auth is required for pull, ensure vault decryption is available non-interactively in runtime (for example, a root-owned vault password file provisioned by operator).
- If decryption is unavailable during periodic reconcile:
  - keep last-known-good running containers,
  - emit a clear warning,
  - exit non-zero only for the update step (do not break already running healthy services).

---

## 6) Suggested host-infra implementation tasks for Codex

1. Add bootstrap helper function `prompt_and_validate_vault_passphrase()`.
2. Add helper `validate_local_env_repo_access()` using deploy-key `git ls-remote`.
3. Add helper `decrypt_local_env_vault_to_json()` (or YAML parse pipeline) for deterministic key extraction.
4. Add Docker login integration before compose pull/up when credentials are present.
5. Add log redaction checks (no token/passphrase in stdout/stderr).
6. Add docs/runbook entries for operator passphrase handling and recovery.

---

## 7) Failure handling and operator UX

- Wrong passphrase: explain that decryption failed; allow retry.
- Missing `secrets.vault.yml`: fail with path + expected structure hint.
- Missing required keys: fail with schema mismatch message.
- Docker login failed: fail update step and include registry/username context (never token).

---

## 8) Security minimums

- Root-only permissions for vault password material (`0600`).
- `set +x` around sensitive shell sections.
- Token/passphrase must not appear in process list, logs, or committed files.
- Prefer short-lived runtime files in `/run` (tmpfs) over persistent disk.
