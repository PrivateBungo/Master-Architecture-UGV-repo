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

## 5) GitOps host convergence

Host convergence exists to keep operational baseline consistent:

- Tailscale state, firewall policy, Docker daemon config
- Device rules/permissions
- systemd units/timers and runtime hygiene

Pull-mode reconcile (robot-local):

1. Fetch latest Git revision.
2. If desired revision changed, run local Ansible converge.
3. Update container stack deterministically (pinned image tags).
4. Run health checks and keep rollback path to last-known-good revision.

---

## 6) Configuration and identity model

- Canonical robot identity is explicit (e.g., `/etc/ugv-robot-id = robot-042`).
- Profiles/overlays select hardware class and per-robot parameters.
- Non-secret config belongs in repo-managed profiles.
- Secrets should be minimized and injected at provisioning/runtime, not committed.

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

This master architecture references three implementation repos:

1. **Host infra** — `Host-infra-for-UGV-controller`
   - Bootstrap, Ansible converge, systemd reconcile, firewall/network baseline, compose/runtime pinning.
2. **Onboard stack** — container services (`webrtc-service`, `uart-bridge`, and future expanded control-plane services).
3. **Firmware** — `hoverboard-firmware-hack-FOC`
   - Real-time motor control, watchdog/safety enforcement, hardware-near command execution.

This repo documents cross-repo contracts and architecture invariants; domain repos own implementation depth.

---

## 9) Architecture invariants (must not break)

- Remote control sends **intent**, not direct motor electrical commands.
- Onboard supervisor remains the sole motion-command authority in Linux domain.
- Firmware enforces local watchdog and safe-stop behavior.
- Tailscale is the primary access boundary; no public inbound requirement.
- Boot and local safety behavior do not depend on GitHub availability.
- Updates should be deterministic, observable, and reversible.
