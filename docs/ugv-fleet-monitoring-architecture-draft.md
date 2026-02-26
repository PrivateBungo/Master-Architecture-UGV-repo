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


## 10) Platform philosophy fit (solution options overview)

This section provides a **tooling-orientation discussion** (not a final selection) to help evaluate leading approaches against the requirements in this draft.

### 10.1 Evaluation lens used

Each platform family is judged against the highest-priority requirements:
- Mission-critical functional-failure detection (camera/PCB/control-link/container failures).
- Log-first triage capability with useful context.
- Agent-based collection on Raspberry Pi–class hardware.
- 24-hour buffering and priority forwarding of critical events.
- Strong bandwidth isolation so monitoring does not degrade mission traffic.
- Lean single-operator operational workflow.

### 10.2 Observability-SaaS-first platforms (all-in-one commercial stacks)

**Typical design philosophy**
- “Instrument everything, centralize everything, correlate everything.”
- Very strong user experience for dashboards, search, alerting, and incident triage.
- Emphasis on broad coverage (metrics, logs, traces, synthetic checks, APM, etc.) with centralized governance.

**Where this fits well for us**
- Fast path to usable log search and alerting for critical failures.
- Good operator ergonomics for a single person managing incidents.
- Strong built-in correlation patterns (service restarts + error logs + host pressure).

**Where this can be squeezed for us**
- Philosophy can trend toward high data ingestion by default, which can conflict with strict bandwidth-isolation requirements unless carefully constrained.
- Cost and complexity can grow if full “collect everything” posture is not actively limited.
- Some advanced features may be unnecessary for current single-operator scope.

### 10.3 Metrics-first cloud-native stacks (Prometheus-style ecosystem + log add-ons)

**Typical design philosophy**
- Open standards and composable components; strong control over what is collected and how.
- Metrics and rule-based alerting first; logs often integrated as a separate but connected subsystem.
- Engineering-driven flexibility over turnkey UX.

**Where this fits well for us**
- Good match for lean control and “collect only what matters.”
- Can be tuned to low-resource edge agents and strict data budgets.
- Strong for explicit prioritization logic and custom failure-rule modeling.

**Where this can be squeezed for us**
- More integration effort to achieve polished log-centric incident workflows out of the box.
- Single-operator teams may feel operational overhead if too many moving parts are introduced early.
- Without discipline, “DIY flexibility” can slow time-to-value.

### 10.4 Log-analytics/SIEM-oriented platforms

**Typical design philosophy**
- Treat logs/events as primary truth; powerful indexing, parsing, and query capabilities.
- Strong event-driven detection workflows and pattern matching.
- Often built with security/forensics roots, then expanded to operations use cases.

**Where this fits well for us**
- Excellent alignment with log-first detection of functional failures.
- Powerful parsing for specific signatures (camera not found, PCB link failure, container startup errors).
- Useful historical investigation and timeline reconstruction.

**Where this can be squeezed for us**
- Can encourage high-volume log ingestion unless tight filters and sampling are enforced.
- May require extra effort to represent low-level host/container health in an operations-friendly way (depending on platform shape).
- If security-centric workflows dominate, operator UX for lean ops may require simplification.

### 10.5 Hybrid edge-first + central SaaS control-plane patterns

**Typical design philosophy**
- Decide/reduce at the edge, escalate centrally: local filtering, buffering, and prioritization before uplink.
- Central plane used for policy, alerting, and cross-fleet visibility.
- Designed around intermittent connectivity and constrained environments.

**Where this fits well for us**
- Very strong fit for war-zone connectivity realities and 24-hour buffering constraints.
- Naturally supports “critical events first” forwarding and mission-traffic protection.
- Aligns with Raspberry Pi–class resource constraints when tuned carefully.

**Where this can be squeezed for us**
- Requires careful edge-policy design to avoid missing useful but non-critical context.
- More up-front design effort in agent behavior, queue priority classes, and drop policies.
- Operational debugging can span edge and cloud boundaries if tooling is fragmented.

### 10.6 Practical fit summary for current phase

For the current phase, the strongest philosophical fit is generally a **log-first, edge-aware, agent-based model** with strict policy controls on bandwidth and buffering:
- Prioritize deterministic detection for mission-blocking failure signatures.
- Keep host metrics minimal and supporting.
- Enforce edge prioritization so critical failures are never displaced by low-value telemetry.
- Keep operator workflow compact (single primary console, minimal states, low alert noise).

### 10.7 Selection criteria to carry into vendor/tool evaluation

When moving from architecture to product selection, require evidence that a candidate can:
1. Run lightweight on Raspberry Pi–class hardware with bounded resource use.
2. Provide robust local buffering and priority queues with a 24-hour policy target.
3. Support hard traffic controls so monitoring cannot crowd out mission traffic.
4. Detect and alert on known failure signatures with low noise and strong context.
5. Operate effectively for a single-operator model today, while allowing gradual scale later.
