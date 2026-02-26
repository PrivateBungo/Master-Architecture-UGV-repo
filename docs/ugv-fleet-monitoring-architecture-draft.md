# UGV Fleet Monitoring Architecture Draft (Lean v2)

## 1) Purpose and scope

This document defines a **lean, operations-first monitoring architecture baseline** for a UGV fleet.

Primary intent:
- Monitor **functional failures** that block or degrade mission capability.
- Monitor **host/device condition signals** only where they help explain functional failures.
- Use a **SaaS-oriented model** with an **agent-based collection approach** on each UGV.

This version intentionally avoids naming specific products or vendors.

## 2) Operating assumptions (current reality)

- The current operating model is effectively **single-operator** (one primary administrator/operator).
- UGVs may be powered on/off unpredictably in a war-zone environment.
- Device offline/online state is often expected and **not, by itself, a high-priority incident**.
- Hardware profile is constrained (assume class similar to **Raspberry Pi 4**).
- Monitoring must remain lightweight and must not degrade core operational functions.

## 3) Monitoring outcomes (what success looks like)

The architecture should enable rapid detection and triage of high-impact failures, especially:
1. Camera/device interface failures (e.g., camera not found).
2. Vehicle-control interface failures (e.g., communication failure with UGV PCB/control board).
3. Container/service startup failures or crash loops.
4. Persistent service-level errors that prevent mission functionality.

## 4) Functional requirements (prioritized)

### 4.1 Critical failure detection (highest priority)
- FR-1: The system shall detect and surface logs/events indicating **functional mission failures**.
- FR-2: The system shall identify camera initialization and camera availability errors.
- FR-3: The system shall identify failures to communicate with UGV control electronics (e.g., PCB interface errors).
- FR-4: The system shall detect container start failures, restart loops, and unhealthy service states.
- FR-5: The system shall classify these failures as **critical** and surface them with high visibility.

### 4.2 Log-centric observability
- FR-6: The system shall ingest logs from host OS and runtime containers required for mission services.
- FR-7: The system shall support filtering/search by UGV, container/service, time, and severity.
- FR-8: The system shall preserve local context around failures (preceding/following log lines/events).
- FR-9: The system shall support pattern-based detection for known failure signatures.

### 4.3 Lean host condition monitoring (supporting context)
- FR-10: The system shall collect minimal host health telemetry relevant to troubleshooting (CPU, memory, disk pressure, basic network quality).
- FR-11: Host telemetry collection shall remain lightweight for low-resource hardware.
- FR-12: Host condition alerts shall be secondary to functional-failure alerts unless they directly impair mission services.

### 4.4 Alerting and response for a single-operator model
- FR-13: The system shall provide a simple severity model (e.g., critical, warning, info).
- FR-14: The system shall reduce duplicate/noisy alerts through deduplication/grouping.
- FR-15: Each alert shall include enough context for immediate triage by one operator.
- FR-16: The system shall track alert state minimally (open, acknowledged, resolved).

### 4.5 Connectivity and buffering behavior
- FR-17: The system shall tolerate intermittent connectivity and forward buffered data when links recover.
- FR-18: Buffering shall prioritize **critical functional-failure logs/events** ahead of lower-priority telemetry.
- FR-19: Required buffering horizon shall be **up to 24 hours**; longer buffering is out of scope for now.
- FR-20: Under pressure, non-critical data may be sampled or dropped before critical failure signals.

### 4.6 Bandwidth isolation and operational safety
- FR-21: Monitoring traffic shall be controlled so it does not materially impact operational/mission data paths.
- FR-22: The architecture shall support explicit bandwidth management and prioritization policies separating monitoring vs operational traffic.
- FR-23: In constrained links, mission/operational traffic shall take precedence over monitoring traffic.

## 5) Non-functional requirements

### 5.1 Resource and performance constraints
- NFR-1: On-device monitoring footprint shall be suitable for Raspberry Pi–class hardware.
- NFR-2: CPU, memory, and storage overhead from monitoring shall remain bounded and configurable.
- NFR-3: Monitoring should deliver critical failure visibility with low practical delay after connectivity is available.

### 5.2 Reliability under field conditions
- NFR-4: Collection should continue locally during transient disconnects.
- NFR-5: The architecture should degrade gracefully by prioritizing critical failure signals.
- NFR-6: The monitoring pipeline itself should emit health signals (agent alive, queue backlog, dropped records).

### 5.3 Security baseline
- NFR-7: Telemetry transport shall be authenticated and encrypted.
- NFR-8: Access controls should match a small-team/single-operator baseline now, while allowing future expansion.
- NFR-9: Sensitive data in logs should be minimized and governed by retention/access policy.

## 6) Architecture decisions to make next (solution-agnostic)

### 6.1 Agent design and placement
- Define agent responsibilities on each UGV (host metrics, container logs, local filtering, buffering).
- Decide what preprocessing occurs at edge vs centrally.

### 6.2 Failure-signal taxonomy
- Define canonical failure classes for mission-critical incidents (camera, PCB/control link, service start/runtime failures).
- Define severity mapping and required alert payload fields.

### 6.3 Buffering and prioritization policy
- Implement queue priorities so critical failure signals are retained/transmitted first.
- Set and enforce a 24-hour maximum buffering objective.

### 6.4 Traffic isolation strategy
- Define hard controls that prevent monitoring data from competing with mission-critical operational traffic.
- Define behavior during severe bandwidth constraints.

### 6.5 Minimal operator workflow
- Define the single-operator incident flow (detect → triage → acknowledge → resolve).
- Define minimum dashboard/query views needed for daily operations.

## 7) Phased rollout (lean)

### Phase 1: Critical failure visibility first
- Deploy agent-based collection for critical logs/events.
- Implement detection and alerting for camera, PCB/control link, and container startup/runtime failures.
- Provide minimal fleet/service views for triage.

### Phase 2: Stabilize and harden
- Add prioritized buffering/replay with 24-hour horizon.
- Add monitoring traffic controls to protect operational bandwidth.
- Tune noisy alert patterns and failure signature rules.

### Phase 3: Broaden only where needed
- Add additional host/service signals only if they improve incident response outcomes.
- Refine retention and performance settings based on field experience.

## 8) Explicitly out of scope for now

- Multi-layer enterprise operating models (complex on-call chains, large admin role matrices).
- Long-horizon buffering beyond 24 hours.
- Fleet-growth forecasting and advanced cost-optimization modeling.
- Broad observability expansion not tied to operational incident response.

## 9) Definition of done for this architecture stage

This stage is complete when:
- Critical failure classes and detection requirements are approved.
- Agent responsibilities and bandwidth-isolation controls are agreed.
- 24-hour buffering policy and prioritization rules are defined.
- Single-operator workflow and minimum views are documented.
