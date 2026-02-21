# UGV Platform Master Architecture

This repository is the **public, cross-repo architecture source-of-truth** for the UGV platform.
It explains how the system fits together across host infrastructure, onboard services, and embedded control without duplicating implementation-level docs from each repo.

---

## 0) Goal and non-goals

### Goals

Build a remotely operable robotic platform (UGV/boat/etc.) that:

- Uses secure private networking (Tailscale) as the access boundary.
- Supports browser-based operation (video + command + telemetry).
- Avoids dependency on custom always-on backend infrastructure.
- Is modular and container-based on onboard compute.
- Preserves hard real-time safety in firmware.
- Scales to many robots and operators with reproducible deployment.
- Uses Git as source-of-truth for host and app delivery.

### Non-goals

- Remote operators do **not** drive PWM directly.
- No public inbound ports are required for operation.
- Robot runtime should not depend on a cloud “brain” to remain safe/functional.

---

## 1) System domains

The platform is split into four domains:

1. **Operator Interface Domain**
2. **Secure Connectivity Domain (Tailnet)**
3. **Onboard Compute Domain (Debian + Containers)**
4. **Embedded Control Domain (Firmware + Motor Control)**

Core principle: **remote intent, local control**.

---

## 2) End-to-end architecture flow

### A. Operator interface (browser)

The operator uses a browser to:

- View low-latency video (WebRTC)
- Send drive intent (steering/speed/mode)
- Monitor telemetry and health state

The UI produces high-level **IntentCommand** messages and consumes video/telemetry.

### B. Secure identity network (Tailscale)

- Robot and operator both join the tailnet.
- ACLs + tags define access control.
- Inbound access is private-by-default (no public port exposure).

Typical traffic over Tailscale:

- WebRTC signaling
- WebSocket control + telemetry
- Optional restricted engineering access (Tailscale SSH)

### C. Onboard host (Debian)

The host is the stable appliance layer and is responsible for:

- `tailscaled` lifecycle (independent of containers)
- Docker/Compose runtime
- Device access (`/dev/video*`, `/dev/tty*`, CAN, etc.)
- Firewall baseline
- Deterministic boot behavior

### D. Onboard container stack (soft real-time)

Typical service responsibilities:

- **webrtc-gateway**: media capture and browser video path
- **control-api**: intent ingress, schema checks, rate limiting
- **supervisor**: command authority, mode machine (`DISARMED`, `ARMED`, `FAILSAFE`, `E-STOP`), shaping/limits, liveness checks
- **telemetry-bridge**: UART/CAN protocol transport + telemetry uplink
- **logger**: structured logs / ring-buffer black box

Optional autonomy modules (e.g., ROS2) are additive, not foundational.

### E. Linux-to-firmware boundary (UART/CAN)

This is a strict protocol contract:

- Linux sends bounded setpoints + mode transitions at fixed rate.
- Firmware returns measurements, status, watchdog, and faults.
- If command stream is stale, firmware must safe-stop.

### F. Firmware execution (hard real-time)

Firmware owns deterministic control and local safety:

- High-rate motor control loop(s)
- Command/watchdog handling
- Setpoint bounds and sanity checks
- Fault detection/latching and safe-stop behavior

**Invariant:** remote command paths cannot bypass firmware safety.

---

## 3) Authority and separation of concerns

To keep the system reviewable and safe:

- **Supervisor is the sole authority issuing motion commands** from onboard compute.
- `control-api` accepts/normalizes operator intent but does not own motion authority.
- `telemetry-bridge` transports protocol data; it does not invent control policy.
- Firmware can always override by transitioning to safe-stop on fault/comms loss.

---

## 4) Deployment model (fleet-ready)

Architecture is intentionally split into:

- **Host layer** (Debian + system services): stable, appliance-like
- **Application layer** (containers): fast iteration

Deterministic boot expectation:

- Robot becomes operational locally even if WAN or GitHub is temporarily unavailable.
- Networking, tailscale, docker, and stack startup are systemd-managed.

---

## 5) GitOps host convergence (robot-local pull mode)

Host convergence exists to keep operational baseline consistent:

- Tailscale state, firewall policy, Docker daemon config
- Device rules/permissions
- systemd units/timers and runtime hygiene

Host-infra implementation model (`ugv-robot-os`) is:

- Canonical robot identity file: `/etc/ugv-robot-id`
- Repo checkout path on host: `/opt/ugv/ugv-robot-os`
- Compose runtime path: `/opt/ugv/stack`
- Reconcile script path: `/usr/local/sbin/ugv-reconcile.sh`
- Reconcile cadence: `ugv-reconcile.timer` every 10 minutes

Pull-mode reconcile flow:

1. Fetch latest Git revision.
2. If desired revision changed, run local Ansible converge (`ansible/site.yml`).
3. Run compose pull/up based on explicit policy (`/etc/default/ugv-reconcile`):
   - default behavior: compose update only when files under `compose/` changed.
4. Run health checks (`tailscaled`, `docker`, and `docker compose ps`).
5. Reboot only when both conditions are true:
   - `/var/run/reboot-required` exists, and
   - `/etc/ugv-allow-reboot` exists.

Update and rollback policy:

- Default Git update mode is fast-forward only.
- Optional strict drift overwrite mode is hard-reset.
- Rollback is Git-native by moving `origin/main` back to prior commit.
- Failed applies must preserve last-known-good running containers.

---

## 6) Configuration and identity model

### Canonical identity and host bootstrap seeds

Each robot is provisioned with:

- A unique hostname.
- A unique SSH deploy key that grants least-privilege read access to the required private repositories.
- Local identity marker(s), for example `/etc/ugv-robot-id`.

These seeded values are the root of robot identity in the deployment pipeline.

### Local environment repository model

Alongside the host-infra and app repositories, each robot pulls from a dedicated local environment configuration repository:

- Repository: `https://github.com/PrivateBungo/UGV-local-environment-variables`
- Robot ID from `/etc/ugv-robot-id` is the canonical lookup key.
- This repo stores device-scoped, operational configuration, for example:
  - Camera type and camera tuning
  - Enabled/disabled services and feature flags
  - Physical dimensions/limits and platform constants
  - Local ACL fragments and host-specific policy settings

This keeps robot-specific operational data versioned, reviewable, and separate from shared application logic.

### File-first fallback behavior for new robots

Installer/runtime resolve env config in this order:

1. `robots/<ROBOT_ID>.env`
2. legacy `robots/<ROBOT_ID>/` (backward compatibility)
3. fallback via compatibility pointer `defaults/default.env`

Selection metadata is materialized for observability:

- `/etc/ugv-env-config-path`
- `/etc/ugv-env-config-kind`
- `/etc/ugv-env-config-dir`

If a newly provisioned robot has no dedicated environment file yet, host infra must load the default baseline from the same local-environment repository.

This default profile enables first-boot safety and predictability while the final robot-specific file is prepared.

### Secret handling and future encryption direction

- Non-secret config belongs in Git-managed robot environment files.
- Sensitive material should still be injected via secure provisioning/runtime paths and not committed in plaintext.
- A planned future enhancement is per-robot encrypted environment payloads (for example, encrypting data so only the intended robot can decrypt it with its provisioned identity key material).

---

## 7) Safety model (end-to-end)

Safety is layered:

1. **UI layer**: explicit arm/disarm + fault visibility
2. **Supervisor layer**: liveness timeouts, shaping/limits, failsafe transitions
3. **Firmware layer**: watchdog, bounds checks, fault latching, safe-stop
4. **Hardware layer**: physical E-stop / power-cut path where available

Never-do rules:

- Never allow direct UI-to-UART raw drive control.
- Never bypass local safety state machines.

---

## 8) Repository mapping

This master architecture references these implementation repos:

1. **Host infra** — `Host-infra-for-UGV-controller`
   - Bootstrap, Ansible converge, systemd reconcile, firewall/network baseline, compose/runtime pinning.
2. **Onboard stack** — `Dockers-for-onboard-UGV-Controller`
   - Current baseline is a two-container scaffold:
     - `webrtc-service`: HTTP/WebSocket signaling placeholder + periodic UDP control heartbeat.
     - `uart-bridge`: UDP listener/parsing placeholder with TODO for UART forwarding.
3. **Firmware** — `hoverboard-firmware-hack-FOC`
   - Real-time motor control, watchdog/safety enforcement, hardware-near command execution.

Additional configuration repo:

4. **Local environment variables** — `UGV-local-environment-variables`
   - Operational env scaffolding split by domain (physical platform, camera, UART PCB, ACL).
   - Default template under `defaults/default/` plus planned `robots/<robot-id>.env` expansion.

Required private repositories for deploy-key bootstrap access:

- `git@github.com:PrivateBungo/UGV-local-environment-variables.git`
- `git@github.com:PrivateBungo/Dockers-for-onboard-UGV-Controller.git`
- `git@github.com:PrivateBungo/Host-infra-for-UGV-controller.git`

This repo documents cross-repo contracts and architecture invariants; domain repos own implementation depth.

---

## 9) Provisioning and first-boot workflows

Two bootstrap paths are currently defined in host infra:

1. **Public seed bootstrap script** (copy/paste self-contained script) that:
   - generates per-repo deploy keys,
   - validates GitHub access repo-by-repo,
   - clones/updates required repositories,
   - invokes host installer with robot ID + env-config repository.
2. **USB first-boot wrapper** (`bootstrap/first-boot-from-usb.sh`) that:
   - consumes `ugv-bootstrap.env` + `ugv-secrets.env`,
   - sets deploy-key Git access and optional infra-repo SSH fallback,
   - runs installer and clears one-time secrets from process environment.

Tailscale auth modes during bootstrap:

- `interactive` (default): operator completes one-time auth URL flow.
- `authkey`: bootstrap uses `TS_AUTHKEY` non-interactively.

---

## 10) Cross-repo implementation guides

To operationalize the above model, use these companion instructions:

- `docs/host-infra-local-env-integration.md` — implementation plan for the host infrastructure repo to clone/select/fallback robot environment files.
- `docs/local-env-repo-scaffold.md` — initial scaffold and conventions for the currently empty `UGV-local-environment-variables` repository.

---

## 11) Architecture invariants (must not break)

- Remote control sends **intent**, not direct motor electrical commands.
- Onboard supervisor remains the sole motion-command authority in Linux domain.
- Firmware enforces local watchdog and safe-stop behavior.
- Tailscale is the primary access boundary; no public inbound requirement.
- Boot and local safety behavior do not depend on GitHub availability.
- Updates should be deterministic, observable, and reversible.
