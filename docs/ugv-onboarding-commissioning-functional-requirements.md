# UGV Commissioning & Onboarding Strategy — Functional Requirements (Draft)

## 1) Purpose

Define functional requirements for a new commissioning workflow where a UGV can be securely onboarded without requiring the platform owner to be physically present at the vehicle, without manual SSH key exchange over CLI, and with an explicit human-verification gate before trust is finalized.

This document is intentionally limited to **functional requirements** and does not prescribe final implementation choices.

---

## 2) Problem statement (current-state pain points)

The current setup flow has the following operational limitations:

1. Commissioning depends on one specific person with privileged repository access.
2. Initial trust material is shown in headless CLI output, which is impractical for remote operators.
3. Setup requires multiple manually handled secrets/keys, creating scalability and handoff friction.
4. Commissioning cannot be cleanly delegated to a field operator without out-of-band key handling.

---

## 3) Scope

### In scope

- First-time trust establishment of a newly provisioned UGV.
- Delegated field commissioning by a remote operator.
- One-time enrollment credential (“seed”) driven onboarding flow.
- Human verification step before a UGV is granted production trust.
- Lifecycle actions tied to onboarding trust (approve, reject, expire, rotate/re-seed).

### Out of scope (for this draft)

- Exact cryptographic primitive/protocol selection.
- Vendor-specific CI/CD or identity-provider integrations.
- UI/UX design details.
- Detailed threat model and compliance controls.

---

## 4) Actors

- **Platform Owner/Admin**: defines trust policy and can approve/reject enrollments.
- **Field Operator/Commissioner**: physically present at UGV; executes setup steps.
- **UGV Node**: device being commissioned.
- **Trust Service**: service that receives enrollment requests and enforces approval workflow.

---

## 5) Functional requirements

### FR-001 — Seed-based bootstrap initiation
The system shall support commissioning a new UGV using a **single bootstrap seed** (or equivalent one-time enrollment token) provided at setup time.

### FR-002 — No manual CLI key transcription
The commissioning flow shall not require operators to manually copy SSH public keys (or equivalent long cryptographic material) from a headless CLI into external systems.

### FR-003 — Delegated commissioning
The system shall allow a field operator to complete technical commissioning steps without requiring direct possession of the platform owner’s personal repository credentials.

### FR-004 — Enrollment request generation
Upon seed use, the UGV shall generate an enrollment request to the trust service that includes: seed identifier, device identity metadata, and commissioning context sufficient for approval decisions.

### FR-005 — Pending trust state
A newly enrolling UGV shall remain in a **pending/untrusted** state until explicit approval is completed, and the pending request shall remain bound to the specific seed identifier used for enrollment.

### FR-006 — Human verification gate
The system shall provide a human verification/approval action prior to final trust activation for a newly enrolled UGV.

### FR-007 — Explicit approve/reject actions
The trust workflow shall support explicit **approve** and **reject** outcomes for each enrollment request.

### FR-008 — Bounded seed validity
Bootstrap seeds shall support bounded validity (e.g., expiration or one-time-use semantics) so stale or leaked seeds cannot be reused indefinitely.

### FR-008a — Human-enterable seed format
The onboarding flow shall support a human-enterable bootstrap seed format suitable for field use (e.g., 16-character/16-digit code), because final trust is protected by a separate human approval gate.

### FR-008b — Seed legitimacy pre-filter
The trust service shall verify that an incoming seed is legitimate (exists, active, and not revoked/expired) before creating a visible pending enrollment request.

### FR-008c — Invalid-seed request handling
Requests tied to invalid, expired, or revoked seeds shall be rejected/ignored and shall not enter the actionable admin approval queue.

### FR-009 — Re-seed and recovery workflow
If seed compromise is suspected, the system shall support issuing a new seed and re-attempting enrollment without requiring manual SSH key exchange.

### FR-010 — Minimal privilege grant at enrollment
Before approval, the UGV shall only receive the minimum access required to complete enrollment checks; production repository/runtime privileges shall only be granted after approval.

### FR-011 — Commissioning status visibility
The system shall expose commissioning status states (at minimum: not started, pending verification, approved, rejected, expired, failed) to authorized operators.

### FR-011a — Seed-scoped queue filtering
The pending-request queue shall support filtering by seed identifier/key so admins can review all requests associated with a specific seed.

### FR-011b — Queue refresh after seed distrust
If an admin revokes/distrusts a seed, queue refresh shall remove or invalidate all pending requests associated with that seed.

### FR-012 — Auditable enrollment events
The system shall record auditable events for seed issuance, enrollment attempts, approvals, rejections, expirations, and re-seeding actions.

### FR-013 — Operator-assisted verification evidence
The workflow shall support attaching or presenting verification evidence (e.g., device serial, location note, visual confirmation artifact) as part of human approval.

### FR-014 — Multi-device scalability
The onboarding workflow shall support commissioning multiple UGVs concurrently without requiring per-device custom manual key trust actions by the platform owner.

### FR-015 — Revocation after onboarding
The system shall provide a mechanism to revoke trust for an onboarded UGV and return it to a non-trusted or re-enrollment-required state.

### FR-016 — Idempotent commissioning behavior
Repeated execution of onboarding steps with the same valid seed and same device shall not create duplicated trusted identities.

### FR-017 — Secure handoff boundary
The field operator shall be able to perform the commissioning procedure without gaining access to long-lived owner-level credentials or global admin secrets.

### FR-018 — Offline-tolerant initiation, online finalization
The commissioning flow shall allow local initiation steps on the UGV when disconnected, while requiring trust-service connectivity for final approval and trust activation.

### FR-019 — Pending window duration
Pending enrollment requests shall expire after a maximum of **48 hours** if no approval action is taken.

### FR-020 — Approval-state retrieval by UGV
The UGV shall retrieve approval/rejection status from the trust service via a defined status-check mechanism (e.g., polling) until terminal state is reached.

---

## 6) Functional architecture components and responsibilities

### C1) Bootstrap Input Mechanism (on UGV)
Responsible for collecting operator-provided bootstrap input and initiating enrollment.

Required functions:

- Accept human-enterable seed during setup.
- Perform basic local validation (format/presence).
- Pass seed and device metadata to enrollment client flow.

### C2) Enrollment Client (on UGV)
Responsible for constructing/submitting enrollment requests and tracking status.

Required functions:

- Build enrollment request containing seed identifier and required metadata.
- Submit request to trust service.
- Keep UGV in untrusted/pending state until approved.
- Poll/check enrollment status and apply result (approved/rejected/expired/revoked).

### C3) Seed Registry
Authoritative store for bootstrap seeds and state.

Required functions:

- Store seed records with lifecycle state (active, expired, revoked, used-policy).
- Support issuance, rotation, revocation, and lookup by seed identifier.
- Provide seed validity result used by enrollment pre-filter logic.

### C4) Enrollment Intake & Verification Service
Front-door for incoming enrollment requests.

Required functions:

- Perform seed legitimacy verification against Seed Registry.
- Reject/ignore requests with invalid/expired/revoked seeds.
- Forward only legitimate requests into pending queue.
- Bind each pending request to originating seed identifier.

### C5) Pending Approval Queue
Operational queue for human review and decisioning.

Required functions:

- Maintain actionable pending requests only.
- Support filtering by seed identifier/key.
- Support queue refresh that revalidates seed status.
- Drop/invalidate pending items whose seed is no longer trusted.

### C6) Admin Approval Interface
Human action surface for trust decisions.

Required functions:

- Display request metadata/evidence for verification.
- Allow explicit approve/reject actions.
- Allow seed distrust/revocation action.
- Trigger queue refresh/reassessment after trust changes.

### C7) Trust Decision Store & Audit Log
System of record for outcomes and accountability.

Required functions:

- Persist approval/rejection/revocation decisions.
- Persist seed lifecycle actions and queue reassessment events.
- Provide auditable timeline of enrollment and trust changes.

### C8) Trust Activation Delivery
Mechanism that communicates decisions back to UGV.

Required functions:

- Expose approval-state endpoint/mechanism for UGV status checks.
- Return terminal decision tied to request identity.
- Ensure revoked/expired outcomes are propagated and enforce untrusted state.

---

## 7) Functional workflow (reference)

1. Admin issues a bounded-lifetime bootstrap seed for a target commissioning window.
2. Field operator starts UGV setup and provides seed through approved input mechanism.
3. Enrollment Intake verifies seed legitimacy before queueing.
4. If seed is invalid/revoked/expired, request is ignored/rejected and never appears in actionable queue.
5. If seed is valid, request enters pending queue bound to that seed identifier.
6. Admin reviews request metadata/evidence and performs human verification.
7. Admin approves or rejects request, or distrusts a seed.
8. Queue refresh revalidates pending requests; all requests tied to newly distrusted seed are removed/invalidated.
9. UGV polls/checks status and applies terminal outcome.
10. On approval, trust service grants production trust profile; UGV transitions to trusted.
11. On rejection/expiration/revocation/failure, UGV remains untrusted and can be re-seeded.

---

## 8) Decision outcomes captured for this revision

1. Maximum pending window is set to **48 hours**.
2. Dual-control/two-person approval is **not required** at this stage.
3. Factory-reset + re-enrollment behavior is **de-prioritized** for now.
4. Seed-scoped filtering and seed-legitimacy pre-filtering are mandatory.

---

## 9) Acceptance baseline for this draft

This draft is considered functionally accepted when stakeholders agree that:

- Commissioning is no longer tied to owner physical presence.
- Manual CLI-based key trust handoff is eliminated.
- Human verification is a mandatory trust gate.
- Seed compromise has a defined recovery path.
- Seed legitimacy filtering blocks invalid requests from the queue.
- Queue reassessment removes pending requests tied to distrusted seeds.
- The process scales operationally beyond one-off device onboarding.
