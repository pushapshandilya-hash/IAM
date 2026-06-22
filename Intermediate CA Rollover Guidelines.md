# Intermediate CA Rollover Guidelines
### for CI/CD Pipelines

> Guidance for designing and automating safe, predictable intermediate CA rollover in CI/CD environments — covering planned and unplanned rollover, trust-bundle distribution across pipeline surfaces, signing-key protection, and emergency readiness.

| | |
|---|---|
| **Owner** | PKI Operations / Office of the Security Architect |
| **Classification** | Internal |
| **Version** | 2.0 |
| **Last reviewed** | June 2026 |

This guideline applies to both publicly-trusted and internal (private) intermediate CAs used by CI/CD pipelines. Where a requirement applies only to one trust domain, it is marked accordingly. It is guidance, not a formal compliance crosswalk; teams under a specific audit framework (e.g. WebTrust for CAs) should verify against the current text of each standard — see [Appendix A](#appendix-a-standards-mapping-by-section).

### Revision history

| Version | Date | Summary of change |
|---|---|---|
| 1.x | — | Initial guideline: planned/unplanned rollover, trust-bundle centralisation, hard-coded-intermediate removal, key storage. |
| 2.0 | Jun 2026 | Added public-vs-private PKI framing and the 47-day TLS schedule; new CI/CD trust-distribution section; planned root rollover and cross-signing; explicit overlap-duration rule; separated trust-bundle removal from revocation; corrected FIPS storage tiers (140-3 foregrounded; Level 3 minimum for issuing-CA keys); added key-ceremony / dual-control, crypto-agility / PQC readiness, a RACI, and emergency-authorisation guidance. |

---

## Contents

- [Regulatory and Operating Context](#regulatory-and-operating-context)
1. [Definitions and Scope](#1-definitions-and-scope)
2. [Policy and Design Principles](#2-policy-and-design-principles)
3. [Centralising CA and Trust Distribution](#3-centralising-ca-and-trust-distribution)
4. [Removing Hard-Coded Intermediates](#4-removing-hard-coded-intermediates)
5. [Trust Distribution in CI/CD](#5-trust-distribution-in-cicd)
6. [Intermediate CA Rollover Process](#6-intermediate-ca-rollover-process)
7. [Secure Storage of Intermediate CA Keys](#7-secure-storage-of-intermediate-ca-keys)
8. [Detection, Validation, and Alerting](#8-detection-validation-and-alerting)
9. [Roles and Responsibilities (RACI)](#9-roles-and-responsibilities-raci)
10. [Documentation and Testing](#10-documentation-and-testing)
- [Sources](#sources)
- [Appendix A: Standards Mapping by Section](#appendix-a-standards-mapping-by-section)

---

## Regulatory and Operating Context

### Public vs. internal trust domains

Before anything else, classify the intermediate being rolled. The governance, the rollover triggers, and even whether external policy applies all depend on its trust domain:

- **Publicly-trusted intermediates** (those that chain to a root in browser/OS root stores) are governed by the CA/Browser Forum Baseline Requirements and the major root programs operated by Mozilla, Google (Chrome), Microsoft, and Apple. These programs can require rapid distrust or revocation with far less notice than a planned cycle.
- **Internal (private) intermediates** used for service-to-service mTLS, artifact signing, or non-public systems are not governed by the CA/Browser Forum; the organisation sets its own validity and policy. Root-program distrust is not a trigger for these, but key compromise, internal policy change, algorithm migration, and CA platform migration still are.

Most CI/CD pipelines touch both: a public intermediate for externally-facing TLS and one or more private intermediates for internal mTLS and signing. Treat them under the same mechanical process below, but apply public-only requirements (root-program monitoring, BR-conformant profiles) only to the public ones.

### Planned vs. unplanned rollover

Intermediate CAs are not on the same validity-reduction schedule as end-entity certificates, but publicly-trusted ones are subject to root-program policies that can force rapid distrust. The practical effect is two modes:

- **Planned rollover** (3–5 year cycle) requires a documented overlap period, staged trust-bundle updates, and pipeline changes tested in non-production ahead of time.
- **Unplanned rollover** (root-program distrust, key compromise, or compliance failure) requires the same steps compressed into hours or days. Organisations that prepare only for planned rollover are routinely caught out by unplanned events.

### The shrinking end-entity lifetime — why automation is now mandatory

Although it does not change intermediate validity, the CA/Browser Forum's Ballot SC-081v3 (approved April 2025) reduces the maximum public TLS leaf lifetime on a fixed schedule: **200 days from 15 March 2026, 100 days from 15 March 2027, and 47 days from 15 March 2029**, with domain-validation reuse falling to 10 days by 2029. This matters directly to rollover for two reasons: shorter leaves mean the overlap period (which must outlast the longest leaf chaining to the old intermediate) shrinks from years toward weeks, making rollover faster and safer; and the renewal frequency makes automated, non-hard-coded issuance non-optional. A pipeline that cannot rotate leaves automatically cannot complete an overlap period on this schedule. (Internal PKIs may set their own leaf lifetimes, but the same automation logic applies.)

### Recommended intermediate CA rollover intervals

| Certificate type | Typical maximum validity (mid-2026) | Recommended rollover target in CI/CD |
|---|---|---|
| Intermediate CA certificate | Commonly 5–10 years for public CAs (per CA/B Forum BRs and root-program policy); organisation-defined for internal CAs | Plan rollover every 3–5 years with a staged transition. Maintain a separate, tested contingency plan for unplanned rollover (§6.5). Size the overlap to the longest-lived leaf under the old intermediate. |

---

## 1. Definitions and Scope

- **Intermediate (subordinate) CA certificate:** certificate for a CA that issues end-entity certificates and is itself signed by a root (or higher) CA.
- **Rollover:** planned or emergency transition from one intermediate (or root) CA to another, keeping both old and new valid and trusted during an overlap period so pipelines and downstream environments stay functional.
- **Trust bundle:** a curated set of trusted root and intermediate CA certificates distributed to CI/CD agents, runtime environments, and verification tools.
- **Cross-signing:** issuing a certificate for the same CA public key from more than one parent (e.g. a new intermediate signed by both the old and new roots) so relying parties on either trust anchor can build a valid path during a migration.
- **Trust anchor:** the root (or pinned intermediate) a relying party treats as implicitly trusted when validating a chain (RFC 5280 §6).
- **Name / EKU / path-length constraints:** technical restrictions placed on a sub-CA (permitted name spaces, extended key usages, and how many further CAs may chain below it) that bound its blast radius.
- **CI/CD pipeline:** automated process that builds, tests, and deploys software and infrastructure.

This guideline covers: safe and predictable intermediate (and, in §6.4, root) CA rollover for both public and internal hierarchies; preparing, switching, and decommissioning intermediates without hard-coded dependencies; distributing trust to the places a CI/CD pipeline actually consumes it; and secure storage and access control for intermediate CA signing keys.

---

## 2. Policy and Design Principles

- **Centralised trust management.** Use a single source of truth (PKI or certificate-management platform) for roots and intermediates; distribute trust bundles from it rather than embedding individual certificates in application repositories.
- **No hard-coded intermediates.** Do not pin specific intermediate certificates in code, images, or configuration. Consume chains and bundles dynamically so rollover needs no code change. This is the single most common cause of rollover outages.
- **Staged rollover with a defined overlap.** Treat intermediate (and root) changes as planned events; ensure every environment trusts both old and new before switching issuance.
- **Crypto-agility and post-quantum readiness.** Treat algorithm and key parameters as variables, not constants. A rollover is the natural moment to change them, so record the key type, size, and signature algorithm of each generation and keep the issuance path able to switch. NIST finalised its first post-quantum standards (FIPS 203/204/205) in August 2024; new intermediates should be provisioned with agility toward that migration in mind, and PQC algorithm validation requires a FIPS 140-3 module (see §7).
- **Continuous monitoring and auditing.** Monitor intermediate validity and chain composition across all environments; alert on approaching expiry, chains to deprecated intermediates, and non-compliant issuers.
- **Readiness for unplanned rollover.** Keep a tested, documented emergency path executable in hours, not weeks; pre-identify the services and pipelines affected by distrust of any given intermediate; test the emergency path at least annually — not just the planned one.

---

## 3. Centralising CA and Trust Distribution

### 3.1 CA and lifecycle system

Designate a primary CA or certificate-lifecycle-management (CLM) system that CI/CD pipelines use — an external CA with API/ACME support, an enterprise CLM platform, or an internal PKI with an automation gateway. The chosen system should:

- Issue certificates and return full chains (end-entity + intermediates + root where needed).
- Publish current root and intermediate bundles.
- Support automated renewal and revocation of intermediate CA certificates.
- Record which intermediate each issued certificate chains to, to support blast-radius analysis during rollover.

### 3.2 Trust bundle management

Define a process for building and distributing standardised trust bundles:

- Include all current roots and active intermediates; during overlap, include both old and new.
- Publish through a single source of truth (configuration or artifact repository, or configuration-management system).
- Consume from CI/CD build agents and runners, runtime environments (containers, VMs, appliances), and any custom verification tools — see Section 5 for how.

Manage trust-bundle updates as a controlled change (pull request and deployment pipeline) so they are auditable and reversible.

---

## 4. Removing Hard-Coded Intermediates

To support rollover, pipelines and applications must not depend on fixed intermediate certificates embedded in configuration. Completing this work before any rollover is a prerequisite; attempting rollover with hard-coded intermediates present will cause trust failures in the environments not yet updated.

- **Inventory usage.** Identify repositories, base images, and config that contain PEM-encoded intermediates, import manually-curated Java keystores or similar, or reference explicit chain files for web servers, proxies, or ingress controllers.
- **Refactor usage.** Replace static intermediates with references to the standard trust bundles (§3.2) or platform/system trust stores where appropriate.
- **Fix the deployment model.** Have container images and runtime environments retrieve trust bundles at build or deploy time rather than baking them in at image-creation time, so a rollover requires only a bundle update, not a rebuild of every image.

---

## 5. Trust Distribution in CI/CD

> **Where CI/CD actually consumes trust.**

Rollovers break in the specific places a pipeline reads trust. Centralised bundles (§3) are necessary but not sufficient: each surface below has its own trust store and its own update mechanism, and a rollover must reach every one before issuance switches. This section is the consumption playbook the rest of the document depends on.

### 5.1 Trust consumption surfaces

| Surface | How trust is consumed | What a rollover requires here |
|---|---|---|
| CI build agents / runners | OS CA store; job-level env vars | Updated bundle delivered before the job runs (not baked into a long-lived runner image) |
| Containers / images | OS ca-certificates layer or mounted bundle | Mount/refresh the bundle at deploy time; avoid baking it into the image |
| Language runtimes | Per-language trust store (see §5.2) | Update or point the runtime at the new bundle; rebuild only if baked in |
| Web servers / proxies / ingress | Explicit chain / CA files | Replace referenced chain file from the bundle source; reload |
| Service mesh / workload mTLS | Mesh trust bundle / SPIFFE bundle | Roll the mesh trust bundle ahead of issuing-CA change (see §5.4) |

### 5.2 Language and platform trust stores

These are the most frequent silent failures, because each ecosystem keeps its own store and ignores the others:

- **OS store:** Linux `ca-certificates` (`update-ca-certificates`), Windows certificate store, macOS keychain.
- **Java:** the JVM `cacerts` / a custom truststore via `keytool` — manually-curated truststores are a classic hard-coded-intermediate trap.
- **Node.js:** the bundled CA list, overridable via `NODE_EXTRA_CA_CERTS`.
- **Python:** `certifi` (and requests/urllib3), overridable via `SSL_CERT_FILE` / `REQUESTS_CA_BUNDLE`.
- **Go:** the system roots, overridable via `SSL_CERT_FILE` / `SSL_CERT_DIR`.

Where possible, point these at the single bundle source rather than maintaining per-language copies.

### 5.3 Containers and ephemeral runners

Deliver the bundle at deploy time — via a mounted ConfigMap/secret, an init container, or a sidecar — not baked into the image at build time. Ephemeral runners should fetch the current bundle on start so a rollover does not require rebuilding every runner image.

### 5.4 Kubernetes and service mesh

In Kubernetes, distribute trust with a purpose-built mechanism (e.g. cert-manager with trust-manager Bundles) rather than hand-rolled ConfigMaps. In a service mesh (Istio, Linkerd) or a SPIFFE/SPIRE deployment, the issuing intermediate is the workload trust anchor; the mesh/SPIFFE trust bundle must carry both old and new intermediates throughout the overlap, and the mesh's own root/intermediate rotation procedure must be sequenced ahead of any issuing-CA change so workloads never see a chain they cannot validate.

### 5.5 Automated issuance (ACME)

Automated issuance (ACME or equivalent) is what makes the overlap period actually complete: leaves must rotate on their own so that, after the issuance switch, certificates chaining to the old intermediate age out without manual intervention. Under the 47-day schedule this is mandatory, not optional — manually-renewed long-lived leaves will keep an old intermediate alive indefinitely and block decommissioning.

---

## 6. Intermediate CA Rollover Process

Implement rollover as a staged process with go/no-go gates and a defined rollback at each gate, plus a separate faster path for unplanned events (§6.5).

### 6.1 Preparation

- **Create the new intermediate.** Generate the new intermediate CA key pair inside the HSM (§7) under a key ceremony with dual control, and sign it with the existing root. Provision it with appropriate technical constraints — name constraints, path-length basic constraints, and EKU restrictions — and record its key algorithm, size, and signature algorithm (§2, crypto-agility). For public intermediates, ensure the profile conforms to current CA/B Forum BRs and root-program expectations for constrained sub-CAs.
- **Configure issuance.** Configure the CLM system to issue under the new intermediate via a new or updated issuance profile — but do not switch yet.
- **Update trust stores.** Update bundles to include existing root(s), existing intermediate(s) — do not remove the old one yet — and the new intermediate(s). Roll the updated bundle to every surface in Section 5 before any issuance switch.
- **Verify.** Run connectivity and chain-validation tests confirming clients in all environments validate chains including the new intermediate. Do not proceed to §6.2 until validation passes in every environment.
- **Rollback gate.** If verification fails anywhere, stop: the only change so far is additive (new intermediate added to bundles), so revert by removing the new intermediate from bundles and investigating before retrying.

### 6.2 Switching issuance to the new intermediate

- **Switch the profile.** Update the CLM issuance profiles used by CI/CD so newly-issued certificates chain to the new intermediate. Pipelines request certificates exactly as before; new certificates chain to the new intermediate with no pipeline code change — the payoff of §2 and §4.
- **Observe the overlap rule.** The overlap period MUST last at least as long as the longest-lived leaf certificate still chaining to the old intermediate. Certificates issued before the cutover keep chaining to the old intermediate and remain valid until they expire or are rotated; certificates issued after chain to the new one. Validation succeeds everywhere because both intermediates are in the bundles. Shorter leaf lifetimes (and ACME auto-renewal, §5.5) shorten this window.
- **Rollback gate.** If post-switch issuance misbehaves, revert the issuance profile to the old intermediate (still present in bundles and its key retained per §7.4) while you investigate; no trust change is needed because both remain trusted.

### 6.3 Decommissioning the old intermediate

Distinguish two different acts, only the first of which belongs in a planned rollover:

- **Remove from trust bundles (planned).** Only after the overlap has fully elapsed and the inventory (§8) confirms no active certificate still chains to the old intermediate: confirm issuance profiles no longer reference it, remove it from bundles, and roll the updated bundles to every surface through the same controlled-change process as §6.1. Then mark it decommissioned in PKI documentation and runbooks.
- **Revoke via the parent (compromise only).** Revoking the old intermediate through the root's CRL invalidates every leaf still chaining to it immediately. In a planned rollover, do NOT revoke until the inventory is clear — removal from bundles is enough. Reserve revocation for compromise or compelled distrust (§6.5), where breaking the affected leaves is the intended outcome.

### 6.4 Planned root rollover and cross-signing

Rolling a root is a longer, planned campaign — not just an emergency case — because relying parties must be migrated to a new trust anchor. The mechanism that keeps it non-breaking is cross-signing:

- **Establish the new root and cross-sign.** Stand up the new root, then issue cross-signed intermediates so the same issuing-CA key chains to both the old and new roots. Relying parties trusting either root can build a valid path during migration.
- **Distribute the new root first.** Add the new root to every trust bundle and validate (Section 5) well before relying on it, exactly as in §6.1.
- **Migrate, then retire.** Move issuance to chain to the new root; let leaves and intermediates under the old root age out; remove the old root from bundles only once inventory confirms nothing depends on it. For public roots, sequence this against root-program inclusion/removal timelines.

### 6.5 Emergency / unplanned intermediate or root rollover

Some events allow only hours or days — a root or intermediate distrusted by a root program, or a CA compelled to revoke for key compromise or compliance failure. To reduce exposure:

- **Maintain readiness, not just process.** Keep a tested emergency path for trust-bundle updates executable in hours, not the weeks assumed by §6.1–6.3. Pre-identify which services and pipelines a given distrust would affect, using the inventory (§8.2), so impact can be scoped immediately.
- **Decouple issuance from a single CA where feasible.** For critical services, evaluate a secondary issuing CA (ideally cross-signed, §6.4) so a single root/intermediate distrust does not block all issuance.
- **Monitor root-program announcements as operational signals.** Track distrust notices (Mozilla, Chrome, Microsoft, Apple) operationally, not just for compliance — these can force a rollover outside the normal cycle. (Public hierarchies only.)
- **Confirm revocation responsiveness.** An emergency revocation is only effective if relying parties promptly receive updated status. Verify CRL publication freshness as the primary signal (and OCSP responder health where OCSP is still in use — note the industry shift away from OCSP toward CRLs and short-lived certificates).

---

## 7. Secure Storage of Intermediate CA Keys

Where the intermediate's private key lives matters as much as how the rollover is orchestrated: a signing key in a world-readable file undermines the whole hierarchy regardless of process quality.

### 7.1 Choosing the right storage tier

Intermediate CA signing keys require hardware-backed, non-exportable protection. Note that FIPS 140-2 validations move to the NIST CMVP Historical List on 21 September 2026 and FIPS 140-3 is required for new validations (and is the only path under which post-quantum algorithms can be validated), so target 140-3 and treat 140-2 as legacy-equivalent. The levels below are aligned across both versions (140-3 Level 3 corresponds to 140-2 Level 3, **not** Level 2).

| Storage option | Key protection | Use for intermediate / root CA signing keys |
|---|---|---|
| Software-backed secrets store | Software-protected keys (FIPS 140-3 / 140-2 Level 1) | Not acceptable for production intermediate CA private keys. Non-production or test hierarchies only. |
| HSM-backed key store (incl. cloud managed HSM) | Hardware-protected, non-exportable keys (FIPS 140-3 Level 3, equivalently FIPS 140-2 Level 3, or Common Criteria EAL 4+) | Minimum acceptable tier for production intermediate CA signing keys. Keys generated and held in hardware; never exportable in plaintext. |
| Dedicated single-tenant HSM | FIPS 140-3 Level 3, single-tenant, dedicated cryptographic hardware | Recommended for production intermediate and root CA signing keys, especially under PCI DSS / HIPAA or where the issuing scope is broad. |

> **Key point.** Storing only the certificate (the public portion) in a generic secrets store is fine. The **private key** must reside in an HSM-backed or dedicated-HSM tier (FIPS 140-3 Level 3 / 140-2 Level 3 / CC EAL 4+) and must never be exportable in plaintext. FIPS 140-2 Level 2 is below the bar for an issuing-CA key.

### 7.2 Access control and network security

- **RBAC with least privilege.** The identity that manages CA lifecycle operations must be separate from the identity that reads or uses issued certificates. Restrict access to signing keys to a very small, audited set.
- **Short-lived credentials.** Use workload identity federation or platform-managed identities so automation interacting with the key store carries no long-lived static secret.
- **Restrict network access.** Confine the HSM / key store to private endpoints and firewall rules scoped to the systems authorised to request signing.
- **Versioning and deletion protection.** Enable on every store holding CA key material; without it, accidental or malicious deletion can destroy the key needed to maintain trust during an overlap.

### 7.3 Key ceremony and dual control

Generation and use of an issuing-CA key are high-trust operations and must not rest with one person. Generate each new intermediate key in a witnessed key ceremony, with the generation, activation, and backup of the key requiring m-of-n quorum (split knowledge / dual control). Record the ceremony (participants, attestations, HSM key-attestation evidence) as auditable proof the private key was generated in hardware and never existed outside it.

### 7.4 Operational key hygiene

- Generate the new key inside the HSM at provisioning time rather than importing it, so it has never existed outside hardware.
- Use distinct HSM partitions or key objects per intermediate generation, so retiring an old intermediate needs no reconfiguration of the partition holding the new one.
- Retain — but disable — the old intermediate's key object for the full overlap period so it remains available for emergency rollback (§6.2). Destroy it only after the overlap has elapsed and the §6.3 checklist is complete.

### 7.5 Supporting the rollover process

- **Audit logging.** Forward all CA key-access events to a central log/SIEM; alert on unexpected access patterns and on signing operations outside approved maintenance windows.
- **Backup and recovery.** Take encrypted backups per the HSM vendor's procedure (key-custody escrow or security-domain backup) and test restore before it is needed. For dedicated HSMs, take security-domain backups at provisioning time to allow recovery or migration of the instance.
- **Compliance scope.** Verify the HSM / key-management solution is audited under the frameworks relevant to your organisation (SOC 2, ISO/IEC 27001, PCI DSS, HIPAA) and confirm current scope against the vendor's published documentation.

---

## 8. Detection, Validation, and Alerting

### 8.1 Policy checks in CI/CD

Add pipeline validation steps that:

- Confirm issuers and chain composition comply with policy (only approved roots and intermediates).
- Detect certificates chaining to an intermediate approaching its planned decommission date.
- Fail the pipeline if a certificate chains to an intermediate already removed from trust bundles.

### 8.2 Certificate inventory and monitoring

Maintain an inventory of all certificates issued under each intermediate, so that the blast radius of a distrust event can be scoped immediately, decommissioning decisions (§6.3) are evidence-based, and the emergency pre-identification requirement (§6.5) has a source of truth.

### 8.3 Alerting

Configure alerts for:

- An intermediate approaching its own expiry or planned rollover date — with staged lead-time thresholds (e.g. 90 / 60 / 30 days) so action is taken with margin.
- Issuance from an intermediate that should no longer be active.
- Unexpected access to intermediate CA signing-key material.
- Root-program announcements (Mozilla, Chrome, Microsoft, Apple) — for publicly-trusted hierarchies.

---

## 9. Roles and Responsibilities (RACI)

R = Responsible, A = Accountable, C = Consulted, I = Informed. Define a named on-call authority empowered to authorise an emergency rollover outside business hours.

| Activity | PKI Ops | Platform/DevOps | App teams | Security/Gov |
|---|---|---|---|---|
| Approve planned rollover plan | R | C | C | A |
| Key ceremony: generate & sign new intermediate | A/R | I | I | C |
| Build & distribute trust bundles | A | R | C | I |
| Update CI/CD consumption surfaces (Sec. 5) | C | R | R | I |
| Switch issuance profiles | A/R | C | I | I |
| Verify chain validation across environments | C | R | R | I |
| Authorise emergency rollover | R | C | I | A |
| Remove from bundle / revoke old intermediate | A/R | C | I | C |
| Monitoring & alerting | C | R | I | R |
| Annual emergency-path & restore test | R | R | C | A |

---

## 10. Documentation and Testing

Before executing a live rollover:

- **Document the process:** source of truth for roots/intermediates and who controls each; how bundles are built and distributed; which CI/CD issuance profiles chain to which intermediate and how they change during rollover; where signing keys are stored (HSM tier, access model, ceremony — §7) and who holds access; the planned overlap and decommission timeline; and the emergency path (§6.5), including who may authorise it out of hours.
- **Test in non-production:** simulate introduction of a new intermediate, bundle updates to every surface type (Section 5), issuance switching, removal of the old intermediate, a cross-signed root rollover (§6.4), an expedited/emergency rollover at least annually, and recovery from an HSM backup to confirm the disaster-recovery path works.
- **Review and sign off:** ensure PKI operations, platform, application, and security teams understand and approve the plan before execution.

---

## Sources

- [CA/Browser Forum Baseline Requirements for TLS (intermediate CA governance)](https://cabforum.org/working-groups/server/baseline-requirements/)
- [CA/Browser Forum Ballot SC-081v3 (TLS certificate lifetime reduction to 47 days)](https://cabforum.org/)
- [Mozilla Root Store Policy](https://wiki.mozilla.org/CA/Root_Store_Policy)
- [Chrome Root Program Policy](https://www.chromium.org/Home/chromium-security/root-ca-policy/)
- [Microsoft Trusted Root Program Requirements](https://learn.microsoft.com/en-us/security/trusted-root/program-requirements)
- [Apple Root Certificate Program](https://www.apple.com/certificateauthority/ca_program.html)
- [FIPS 140-3 — Security Requirements for Cryptographic Modules (current standard), NIST](https://csrc.nist.gov/publications/detail/fips/140/3/final)
- [FIPS 140-2 — Security Requirements for Cryptographic Modules (legacy; Historical List Sept 2026), NIST](https://csrc.nist.gov/publications/detail/fips/140/2/final)
- [NIST Post-Quantum Cryptography Standards (FIPS 203/204/205)](https://csrc.nist.gov/projects/post-quantum-cryptography)
- [NIST SP 800-57 Part 1 Rev. 5 — Recommendation for Key Management](https://csrc.nist.gov/publications/detail/sp/800-57-part-1/rev-5/final)
- [RFC 5280 — X.509 PKI Certificate and CRL Profile, IETF](https://www.rfc-editor.org/rfc/rfc5280)
- [RFC 6960 — X.509 Online Certificate Status Protocol (OCSP), IETF](https://www.rfc-editor.org/rfc/rfc6960)
- [RFC 3647 — Certificate Policy and Certification Practices Framework, IETF](https://www.rfc-editor.org/rfc/rfc3647)
- [SPIFFE / SPIRE (workload identity and trust-bundle distribution)](https://spiffe.io/)
- [cert-manager and trust-manager (Kubernetes certificate and trust distribution)](https://cert-manager.io/)

---

## Appendix A: Standards Mapping by Section

Guidance-level mapping of each section to the external standards it draws on — not a formal compliance crosswalk. Teams under a specific audit framework (e.g. WebTrust for CAs) should verify against the current text of each standard.

| Section | Standards / references | Why it applies |
|---|---|---|
| Regulatory context; intervals | CA/B Forum BRs; Ballot SC-081v3; Mozilla/Chrome/Microsoft/Apple root programs | Define public intermediate governance, the leaf-lifetime schedule, and the triggers for unplanned rollover |
| 1. Definitions and scope | RFC 5280 | Defines intermediate / chain / trust-anchor / cross-signing terminology |
| 2. Policy and design principles | NIST SP 800-57 Pt 1; CA/B Forum BRs; NIST PQC (FIPS 203/204/205) | Underpin key-rotation, centralised trust, and crypto-agility / PQC readiness |
| 3. Centralising CA and trust distribution | RFC 5280 §6; CA/B Forum BRs §6.1–6.3 | Govern how roots/intermediates are issued, published, and consumed |
| 4. Removing hard-coded intermediates | RFC 5280 §6 (path validation) | Path validation depends on dynamic chain discovery, not pinned intermediates |
| 5. Trust distribution in CI/CD | RFC 5280; SPIFFE/SPIRE; cert-manager / trust-manager | Practical trust-anchor distribution to pipeline, container, mesh, and workload surfaces |
| 6. Rollover process (incl. cross-signing, emergency) | CA/B Forum BRs; root-program policies; RFC 5280 (path building) | Govern intermediate/root lifecycle, cross-signed migration, and distrust-driven emergencies |
| 7. Secure storage of CA keys | FIPS 140-3 (Level 3) / FIPS 140-2 (legacy); CC EAL 4+; NIST SP 800-57 Pt 1 | Define the cryptographic-module assurance and key-management controls for CA signing keys |
| 8. Detection, validation, alerting | RFC 6960 (OCSP); RFC 5280 §6; CA/B Forum BRs (revocation) | Basis for chain-compliance, revocation, and issuer checks during and after rollover |
| 9. Roles and responsibilities | RFC 3647 (roles in a CP/CPS) | Standard basis for assigning CA operational responsibilities |
| 10. Documentation and testing | RFC 3647 | Standard structure for documenting CA practices and exercising the runbook |

---

*Guidance only, not a formal compliance crosswalk. Verify against the current text of each standard for audit purposes.*
