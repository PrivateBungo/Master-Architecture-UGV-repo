# Scaffold Instructions: `UGV-local-environment-variables` (with Ansible Vault)

This document is for the `UGV-local-environment-variables` repository and is written as a **Codex-ready implementation brief**.

Primary goal: define a predictable repository structure where Docker Hub credentials are stored only in an Ansible Vault file that host-infra can decrypt during bootstrap/reconcile.

---

## 1) Repository purpose

Store robot operational configuration in two layers:

1. Plain non-secret config (`*.env`) for branch/repo/feature defaults.
2. Encrypted secret config (`secrets.vault.yml`) for sensitive values such as Docker Hub token.

No plaintext secrets in committed `.env` files.

---

## 2) Required directory contract

```text
UGV-local-environment-variables/
├─ README.md
├─ defaults/
│  └─ default/
│     ├─ general.env
│     ├─ physical-ugv.env
│     ├─ camera.env
│     ├─ uart-pcb.env
│     ├─ acl.env
│     └─ secrets.vault.yml          # ansible-vault encrypted file
├─ <ROBOT_ID>/
│  └─ env/
│     ├─ general.env
│     ├─ ... other *.env files ...
│     └─ secrets.vault.yml          # optional robot override (encrypted)
├─ schemas/
│  └─ vault-schema.md
└─ scripts/
   └─ validate-env.sh
```

Selection semantics expected by host-infra:

- First choice: `<ROBOT_ID>/env/`
- Fallback: `defaults/default/`

Both locations must follow the same filename contract, especially `secrets.vault.yml`.

---

## 3) Required plaintext file content

`defaults/default/general.env` must include at least:

```dotenv
HOST_INFRA_REPO_BRANCH=main
HOST_INFRA_REPO_SSH=git@github.com:PrivateBungo/Host-infra-for-UGV-controller.git
LOCAL_ENV_REPO_SSH=git@github.com:PrivateBungo/UGV-local-environment-variables.git
DOCKER_STACK_REPO_SSH=git@github.com:PrivateBungo/Dockers-for-onboard-UGV-Controller.git
```

This keeps branch/repo metadata outside vault and aligns with current bootstrap/reconcile expectations.

---

## 4) Required vault schema (predictable + strict)

`secrets.vault.yml` must decrypt to this shape:

```yaml
registry_auth:
  dockerhub:
    username: "<dockerhub-username>"
    token: "<dockerhub-access-token>"
    registry: "docker.io"
```

Rules:

- `registry_auth.dockerhub.username` is required.
- `registry_auth.dockerhub.token` is required.
- `registry_auth.dockerhub.registry` is optional; default assumed `docker.io`.
- No additional top-level keys unless host-infra parser is updated accordingly.

---

## 5) How to create/update vault file

Example workflow for operators/developers:

```bash
cd UGV-local-environment-variables

cat > /tmp/secrets.plain.yml <<'YAML'
registry_auth:
  dockerhub:
    username: "my-dockerhub-user"
    token: "dckr_pat_..."
    registry: "docker.io"
YAML

ansible-vault encrypt /tmp/secrets.plain.yml \
  --output defaults/default/secrets.vault.yml

rm -f /tmp/secrets.plain.yml
```

For robot-specific override:

```bash
ansible-vault encrypt /tmp/secrets.plain.yml \
  --output <ROBOT_ID>/env/secrets.vault.yml
```

---

## 6) Validation requirements

`scripts/validate-env.sh` should verify:

1. Required directories exist (`defaults/default/`).
2. Required plaintext file exists (`general.env`).
3. Required key presence in `general.env`.
4. `secrets.vault.yml` exists in required directories.
5. Vault file starts with Ansible Vault header (`$ANSIBLE_VAULT;`).
6. Optional: schema check by decrypting with CI-provided vault password and validating required keys.

---

## 7) Suggested tasks for Codex in local-env repo

1. Create `defaults/default/` structure and baseline env files.
2. Add encrypted `defaults/default/secrets.vault.yml` using schema above.
3. Add `schemas/vault-schema.md` documenting required keys and meanings.
4. Add `scripts/validate-env.sh` for naming/syntax/required-file checks.
5. Update repo README with operator instructions for vault creation/rotation.

---

## 8) Rotation and operational guidance

- Rotate Docker Hub token by updating plaintext source and re-encrypting vault.
- Commit vault ciphertext only.
- Share vault passphrase out-of-band (never in git).
- If passphrase changes, host operators must provide updated passphrase at bootstrap/reconcile secret provisioning step.
