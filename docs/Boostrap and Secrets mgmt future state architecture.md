# Bootstrap and Secret Management – Future State Architecture

## 1. Purpose & Scope

This document defines the **target architecture** for secure device onboarding ("bootstrap") and secrets management in the UGV fleet.  
It is designed for operational flexibility, security, and clarity in highly distributed deployments, including semi-trusted field conditions.

---

## 2. Goals and Principles

- **Minimize secrets exposure before device provisioning**
- **Human-in-the-loop device registration:** Only approved devices obtain identity and fleet credentials
- **No device can self-register for secrets without manual review**
- **All long-term secrets delivered post-onboarding, always in encrypted form, per device**
- **Compromise of bootstrap credentials or DMZ does not grant production access**

---

## 3. Threat Model

- **Bootstrap (DMZ) network is not trusted:** Devices in this network may be malicious or rogue.
- **Generic VPN credentials are fielded for convenience, but must be easily revocable.**
- **Device manufacturing/distribution can occur outside of secure network.**
- **All secrets are encrypted; plaintext secrets never leave secured Vault.**
- **Operator decides which device becomes which unique UGV identity.**

---

## 4. Components

| Component              | Role                                             | Exposure            |
|------------------------|--------------------------------------------------|---------------------|
| UGV Device (Fielded)   | Requests identity, generates keypair, decrypts   | DMZ/prod            |
| Bootstrap VPN          | Network ingress for new, untrusted devices       | DMZ/field           |
| Provisioning Server    | DMZ service, human approval, brokers onboarding  | DMZ/trusted LAN     |
| Vault Server           | Stores per-UGV secrets, encrypts/publishes blobs | Only trusted LAN    |
| Public Encrypted Repo  | Publishes encrypted configs, public keys         | World-readable      |
| Operator UI            | Assigns identities, approves registrations       | Only trusted users  |

---

## 5. Bootstrap/Provisioning Workflow

### Step-by-step

1. **Image Flashing / Shipment**
    - Each device is flashed with a generic OS + Bootstrap VPN config/profile—nothing device-unique or secret.
    - Generic Bootstrap VPN config may be shipped with each SD card, imaged en masse, or externally loaded (USB).

2. **Initial Connection (DMZ)**
    - UGV device connects to the Bootstrap VPN/DMZ network when first powered on.
    - Device runs a lightweight bootstrap agent that announces itself to the Provisioning Server.

3. **Identity Challenge and Operator Approval**
    - Provisioning Server logs all requesting devices (showing MAC, serial, etc.)
    - Human operator reviews “pending” list, matches device to label/physical context, and clicks to “Assign ID.”
    - Server prompts device to generate a new OpenPGP keypair and upload its **public key only**.

4. **Vault Registration & Secret Seeding**
    - Provisioning Server forwards (ID, public key, device metadata) to Vault.
    - Vault creates new per-device secret/config payload (minimum: device identity, long-term VPN profile, fleet secrets, API keys, etc.)
    - Vault encrypts payload with the UGV’s public key, publishes encrypted blob to Public Encrypted Repo.

5. **Bootstrap Completion**
    - Provisioning Server notifies device that registration is complete.
    - Device is instructed where to fetch its encrypted configuration.
    - Device retrieves, decrypts (using its own private key), and installs secrets locally.
    - Device deletes Bootstrap VPN config, switches to production credentials & long-term role.

---

## 6. Security Advantages

- **Compartmentalization:** Devices cannot access Vault or production assets from DMZ; all sensitive steps require human approval.
- **Revocability:** If bootstrap VPN credential is leaked, only that onramp is replaced (devices already provisioned are unaffected).
- **Auditable Onboarding:** All ID assignments, key registrations, and approvals are logged.
- **No plaintext distribution:** Secrets only ever exist unencrypted on Vault and device after decryption.
- **Easy deployment scaling:** Bulk images can be prepared; uniqueness comes from provisioning process, not up-front.

---

## 7. Sequence Diagram

```
+-----------------+        +---------------------+        +------------+        +---------------------+        +-----------------------+
| UGV Device      |        | Provisioning Server |        | Operator   |        | Vault Server        |        | Public Encrypted Repo |
+-----------------+        +---------------------+        +------------+        +---------------------+        +-----------------------+
        |                          |                          |                      |                            |
        |---Connects to Bootstrap VPN-----------------------> |                      |                            |
        |--Announces self ("who am I?")---------------------> |                      |                            |
        |                          |---Displays pending-----> |                      |                            |
        |                          |                          |--Select "Assign ID"->|                            |
        | <----Requests PGP pubkey--|                          |                      |                            |
        |---Generates keypair, sends pubkey-----------------> |                      |                            |
        |                          |---Forwards info--------->|                      |                            |
        |                          |                          |---Registers---------->|                            |
        |                          |                          |                      |--Encrypts config           |
        |                          |                          |                      |--Publishes encrypted blob->|
        |                          |---Notifies device------->|                      |                            |
        |<---Instructs on fetch/decrypt---------------------- |                      |                            |
        |---Fetches encrypted blob from repo----------------->|                      |                            |
        |---Decrypts locally / installs secrets              |                      |                            |
        |---Deletes bootstrap VPN profile                    |                      |                            |
        |---Switches to production operation                 |                      |                            |
```

---

## 8. Implementation Guidance

- **Bootstrap VPN Profile:** Kept generic, easily revocable/distributable, grants access only to DMZ/provisioning resources.
- **Provisioning Server:** Should enforce rate-limits, validation (to prevent rogue device attacks/flooding).
- **Keypair Generation:** Always prefer generating the private PGP key on-device; never transfer private keys.
- **Operator Workflow:** Consistent UI clearly shows device state, hardware info, and lets operator approve/assign roles. All actions are auditable.
- **Vault Server:** Does not accept requests or connections from DMZ/devices; only from Provisioning Server/operator.
- **Public Encrypted Repo:** Contains only encrypted blobs and public keys—safe to expose.
- **Device Transition:** After bootstrap, device wipes any generic credentials and uses per-device/role secrets only.

---

## 9. Future Extensions

- **Hardware-based identity attestation (TPM, Secure Boot, etc.)** for on-device private key protection.
- **End-to-end onboarding automation** for larger fleets (with "manual override" always available).
- **Remote decommissioning/rotation tools.**
- **Enhanced audit: signed approvals, compliance reporting.**
- **Potential PKI/CA integration for "federated" operator domains.**

---

## 10. Notes on Migration/Security Gap Closure

- Existing processes can be wrapped to support this model incrementally.
- In case devices are already provisioned, enforce re-bootstrap after any global credential compromise.
- Revocation and audit processes should be a part of operational SOPs.

---

## Appendix: Example Directory Layout

**On Vault (private):**
```
vault/
  UGV-001/
    secrets.env
    ugv-001.pub
    config.json
  ...
public-encrypted-repo/
  UGV-001.secrets.env.gpg
  keys/UGV-001.pub
```

**On DMZ/Provisioning Server:**
- Registration scripts, approval logs, device database.

**On UGV:**
- secrets.env (after decryption)
- ugv-001.sec (private key, permissions 0600)

---

## Conclusion

This architecture ensures effective, scalable, and secure onboarding and secret management for UGVs across varied operational and field environments.  
It makes bootstrapping safe to delegate operationally (via generic images/keys), while reserving real trust and secrets exposure for only operator-approved, individually-registered hardware.
