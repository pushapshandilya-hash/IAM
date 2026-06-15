# CI/CD Security Standard

| Field | Value |
|---|---|
| Document type | Security control standard (normative, vendor-neutral) |
| Issued by | Security Architecture |
| Status | Draft for review — not yet ratified |
| Version | 0.4 |
| Supersedes | CI/CD Security Standard / Governance Standard v0.1–0.3 |
| Scope | All CI/CD pipelines and supporting platforms across the organisation's heterogeneous toolchain |
| Review cadence | Quarterly, or on material change to the platform estate or threat landscape |

**Conventions.** Keywords **MUST/MUST NOT** (mandatory), **SHOULD/SHOULD NOT** (recommended; discretionary), and **MAY** (optional) follow RFC 2119. An unmet MUST requires a time-boxed exception (§12); a SHOULD does not.

---

## 1. Purpose

This standard defines the mandatory security controls for all CI/CD pipelines across the organisation's heterogeneous toolchain — multiple source-control and CI/CD platforms serving both internal teams/BUs and external customers/partners.

In a heterogeneous estate, security posture is set by the weakest-governed repository on the least-mature platform. The standard therefore specifies control *intent* once, requires every platform to meet it, and mandates detection of anything below baseline.

It is governed by four principles: **Consistency** (one control intent across all platforms), **Differentiation** (intensity scales with risk tier, §9), **Scalability** (controls inherited by default), and **Manageability** (central policy, delegated implementation).

---

## 2. Scope

**In scope.** All source-control platforms, CI/CD orchestrators, build/execution environments, artifact and secret stores, and deployment controllers used to deliver organisational software; all pipelines that produce a release-able artifact or effect a deployment; all human and non-human identities acting within them; and all tenants served (internal and external).

**Out of scope** (governed elsewhere): developer endpoint security; application runtime security beyond what the pipeline deploys; production infrastructure not provisioned via CI/CD; corporate identity governance (a dependency, §6.2); data classification policy (a dependency, §6.11).

**Applicability.** Applies regardless of hosting model (self-hosted, managed, or third-party-operated). Third-party-operated platforms MUST be brought into conformance contractually.

---

## 3. Definitions

- **Estate** — the complete set of in-scope platforms and their repositories/pipelines.
- **Platform** — any in-scope source-control or CI/CD system.
- **Control plane / data plane** — the CI/CD systems / the workloads they deploy.
- **Pipeline** — an automated sequence from input (commit, trigger) to artifact and/or deployment.
- **Artifact** — any release-able build output.
- **Provenance** — verifiable, signed metadata describing how and from what an artifact was built.
- **Tenant** — an isolated context (code, config, secrets, data, runtime) that must be segregated; *internal* (team/BU) or *external* (customer/partner).
- **Segregation model** — the declared mechanism keeping a tenant's resources separate.
- **Untrusted / trusted pipeline** — operating on un-reviewed input / on reviewed, approved input with privileged downstream access.
- **Baseline** — the MUST controls applying to every repository regardless of tier.
- **Conformance register** — the authoritative record of how each platform implements each control (Appendix A).

---

## 4. Governance and ownership

This standard is owned, issued, and maintained by Security Architecture, which defines the baseline, assigns tiers, ratifies versions, and approves exceptions. Platform owners implement the controls; product teams conform their repositories and pipelines; tenant owners declare segregation requirements. Detailed accountabilities are in §10.

**Inherited by default.** New repositories, pipelines, platforms, and tenants inherit the baseline automatically. Opting *out* of any baseline control requires an exception (§12); opting *in* requires no action.

---

## 5. Control objectives

Each maps to the OWASP Top 10 CI/CD Security Risks (`SEC-n`) it addresses (§15); §6 specifies the satisfying requirements.

| ID | Control objective | Primary risk |
|---|---|---|
| CO-1 | Maintain complete, current visibility of the estate | SEC-10 |
| CO-2 | Ensure identity and access integrity across all platforms | SEC-2 |
| CO-3 | Enforce flow control and change integrity in source | SEC-1 |
| CO-4 | Preserve pipeline execution integrity | SEC-4 |
| CO-5 | Constrain pipeline-scoped access to least privilege | SEC-5 |
| CO-6 | Maintain credential and secret hygiene | SEC-6 |
| CO-7 | Harden and baseline system configuration | SEC-7 |
| CO-8 | Govern dependencies and third-party components/extensions | SEC-3, SEC-8 |
| CO-9 | Guarantee artifact integrity and provenance | SEC-9 |
| CO-10 | Ensure logging, audit, and detection coverage | SEC-10 |
| CO-11 | Enforce tenant segregation (internal and external) | SEC-2, SEC-5, SEC-7 |
| CO-12 | Maintain resilience and controlled emergency change | operational |

---

## 6. Control requirements

Stated as platform-agnostic capabilities; per-platform implementation is recorded in Appendix A.

### 6.1 Estate visibility (CO-1)
- **6.1.1 (MUST)** A continuously maintained inventory of all platforms, organisations/projects, repositories, and pipelines MUST exist. Unknown repositories are presumed non-conformant.
- **6.1.2 (MUST)** New organisations, projects, or repositories MUST be discoverable within a defined detection window.
- **6.1.3 (SHOULD)** Each repository SHOULD carry ownership and tier metadata to enable automatic control application.

### 6.2 Identity and access (CO-2)
- **6.2.1 (MUST)** Human identities MUST authenticate through the central identity provider via SSO. Standalone/local accounts MUST NOT be used for standing access.
- **6.2.2 (MUST)** Joiner/mover/leaver lifecycle MUST be federated so one deprovisioning action removes access across all platforms. Orphaned access on any platform is a reportable defect.
- **6.2.3 (MUST)** Access MUST follow least privilege, be granted to roles/groups, and be recertified periodically.
- **6.2.4 (MUST)** Non-human identities MUST be inventoried, owned, scoped, and lifecycle-managed like human identities.
- **6.2.5 (MUST)** Segregation of duties MUST be enforced: authoring security policy or granting exceptions MUST be separated from deploying to production.
- **6.2.6 (MUST)** External collaborators MUST be provisioned through a governed process with explicit scope and expiry; standing external access MUST NOT be default.

### 6.3 Source control and flow control (CO-3)
- **6.3.1 (MUST)** Production-bound branches MUST be protected: no direct pushes; changes require review by someone other than the author.
- **6.3.2 (MUST)** Pipeline definitions, build configuration, and policy-as-code MUST reside in protected locations under stricter review than application code.
- **6.3.3 (MUST)** Secret scanning MUST run pre-merge; a detected secret blocks the merge and triggers rotation.
- **6.3.4 (SHOULD)** Commit/tag signing SHOULD be enforced; unsigned changes SHOULD NOT be promotable to production.
- **6.3.5 (MUST)** Self-approval and check-bypass MUST be disabled for production-bound changes except via break-glass (§6.13).

### 6.4 Pipeline execution integrity (CO-4)
- **6.4.1 (MUST)** Untrusted pipelines MUST be separated from trusted pipelines so untrusted input cannot obtain production or cross-tenant credentials.
- **6.4.2 (MUST)** Pipelines triggered by un-reviewed external input MUST run without secrets, production identity, or other tenants' resources.
- **6.4.3 (MUST)** Production build environments MUST be isolated per job and MUST NOT persist credentials/state in a way enabling tampering or cross-tenant leakage. Necessary persistent environments MUST be a declared exception with compensating isolation.
- **6.4.4 (MUST)** Build caches and shared resources MUST NOT carry data or artifacts across trust or tenant boundaries.

### 6.5 Pipeline-scoped access (CO-5)
- **6.5.1 (MUST)** Pipeline permissions MUST be scoped to exactly the resources and tenant(s) targeted; a pipeline MUST NOT enumerate or act on out-of-scope resources.
- **6.5.2 (MUST)** Pipeline privilege MUST be time-bound and revocable.

### 6.6 Secret and key management (CO-6)
- **6.6.1 (MUST)** Secrets MUST reside in a managed store — never in source, pipeline definitions, logs, or artifacts.
- **6.6.2 (MUST)** Standing long-lived production/cloud credentials MUST NOT be held by any platform; short-lived federated workload identity MUST be used.
- **6.6.3 (MUST)** Secrets MUST be rotatable; rotation MUST be automated for high tiers (§9).
- **6.6.4 (MUST)** Secrets and keys MUST be scoped so no single credential grants *effective* cross-tenant access (permitting standard hierarchical key schemes).
- **6.6.5 (SHOULD)** Customer-managed key options SHOULD be available where external-tenant tiers contractually require them.

### 6.7 System hardening (CO-7)
- **6.7.1 (MUST)** Each platform MUST be configured to a documented hardened baseline; drift MUST be detectable.
- **6.7.2 (MUST)** Security-relevant configuration MUST be expressed as policy-as-code where supported and enforced automatically.

### 6.8 Dependency and third-party governance (CO-8)
- **6.8.1 (MUST)** External dependencies/base components MUST be sourced from approved, integrity-verified origins and pinned by version/digest.
- **6.8.2 (MUST)** Third-party pipeline components (steps, plugins, extensions, reusable workflows) MUST be allow-listed by source, pinned to an immutable reference, and reviewed before use. Untrusted components MUST NOT run in trusted pipelines.
- **6.8.3 (MUST)** A software bill of materials MUST be produced for every artifact.
- **6.8.4 (SHOULD)** Dependency and component risk SHOULD be continuously monitored against new advisories.

### 6.9 Artifact integrity and provenance (CO-9)
- **6.9.1 (MUST)** Every release-able artifact MUST be signed with signed build provenance (source, inputs, builder identity).
- **6.9.2 (MUST)** Signatures and provenance MUST be verified at deploy/admission, not only at build. Unverifiable artifacts MUST NOT be deployed.
- **6.9.3 (MUST)** Released artifact versions MUST be immutable; mutable references MUST NOT be deployed to production.
- **6.9.4 (SHOULD)** Builds SHOULD be reproducible to support provenance verification.
- Per-tier provenance bars are set in §9 by reference to a recognised supply-chain integrity-level framework.

### 6.10 Deployment and release control (CO-9, CO-11)
- **6.10.1 (MUST)** Tier-appropriate release-approval gates MUST be enforced by the pipeline, not by convention.
- **6.10.2 (MUST)** Deployment MUST use short-lived, scoped, federated identity.
- **6.10.3 (MUST)** For shared/multi-tenant targets, deployment MUST be progressive so one change cannot reach all tenants at once; health gates MUST halt promotion on failure.
- **6.10.4 (MUST)** Automated rollback MUST be available and tested.
- **6.10.5 (MUST)** Before applying a change, the controller MUST validate the target resolves to the intended tenant scope and preserves the declared segregation model; failure blocks deployment.

### 6.11 Tenant segregation (CO-11)
- **6.11.1 (MUST)** Every tenant MUST have a declared segregation model; pipelines MUST enforce its invariants.
- **6.11.2 (MUST, internal)** Cross-team/BU access MUST be denied by default; one team's pipeline MUST NOT act on another's resources without explicit grant.
- **6.11.3 (MUST, external)** Pipelines and credentials delivering to external customers/partners MUST be segregated per customer; no shared credential or resource may grant effective cross-customer access.
- **6.11.4 (MUST)** External-customer or regulated data MUST NOT exist in shared or lower environments without approved de-identification.

### 6.12 Logging, audit, and detection (CO-10)
- **6.12.1 (MUST)** All platforms MUST emit security-relevant audit events to a central, append-only, access-controlled store with tier-defined retention.
- **6.12.2 (MUST)** Events MUST be normalised across platforms to enable estate-wide detection and investigation.
- **6.12.3 (MUST)** Privileged and anomalous control-plane actions MUST be monitored and alertable.

### 6.13 Resilience and emergency change (CO-12)
- **6.13.1 (MUST)** A break-glass procedure MUST exist: additional authorisation, fully logged, time-boxed, with mandatory post-event review.
- **6.13.2 (MUST)** An estate-wide/per-platform deployment freeze ("kill switch") MUST be available to contain suspected compromise.
- **6.13.3 (SHOULD)** Pipeline configuration SHOULD be recoverable as code for disaster recovery.

---

## 7. Heterogeneous-estate conformance

- **7.1 (MUST)** Every applicable §6 requirement MUST be implemented on every in-scope platform using that platform's capabilities. Intent is uniform; mechanism may differ.
- **7.2 (MUST)** Each platform owner MUST maintain a conformance-register entry (Appendix A) per requirement: mechanism and status.
- **7.3 (MUST)** A native gap MUST be recorded with a compensating control and an exception (§12) — never left silent.
- **7.4 (MUST)** Automated posture/drift detection MUST run across the estate. The **weakest-link principle** governs: estate posture is reported at the level of its least-conformant in-scope component.

---

## 8. Lifecycle alignment

| Phase | Controlling sections |
|---|---|
| Plan / Design | §4, §9, tier-based threat modelling |
| Develop / Source | §6.3, §6.6 (pre-merge), §6.8.1 |
| Build | §6.4, §6.8, §6.9.1 |
| Test | §6.8.4, tier-based scanning (§9) |
| Release | §6.9, §6.10.1 |
| Deploy | §6.10, §6.11 |
| Operate | §6.12, §6.13 |

---

## 9. Differentiation: control tiers

Each repository/pipeline and each tenant is assigned a tier; the **higher of the two governs**. External-customer-facing and production-deploying repositories default to higher tiers.

| Control area | Tier 1 (low / internal, non-prod) | Tier 2 (standard prod / internal tenant) | Tier 3 (external-customer / regulated / high-criticality) |
|---|---|---|---|
| Branch protection & review | Required | Required + designated owners | Required + ≥2 reviewers |
| Pre-merge scanning gate | Warn | Block on high | Block on medium+ |
| Dynamic/interactive testing | Optional | Recommended | Required |
| Artifact signing & provenance | Required (basic) | Required + verified at deploy | Higher integrity level + key ceremony |
| Secret rotation | Manual acceptable | Automated | Automated + short TTL |
| Segregation assurance | Logical | Logical + validated at deploy | Validated + independent review |
| Release approval | Automated gate | Automated gate | Manual approval at final stage |
| Key management | Platform-managed | Platform-managed | Customer-managed option |
| Audit retention | Standard | Extended | Compliance-defined |
| Threat model | Lightweight | Required | Required + periodic refresh |

---

## 10. Roles and responsibilities

| Activity | Security Architecture | Platform owner | Product/Eng team | Tenant owner |
|---|---|---|---|---|
| Own and ratify this standard | **A/R** | C | I | I |
| Define baseline & tiers | **A/R** | C | C | C |
| Implement controls on a platform | C | **A/R** | C | I |
| Maintain conformance register | C | **A/R** | I | I |
| Conform repos/pipelines to baseline & tier | I | C | **A/R** | I |
| Estate inventory & drift detection | **A** | R | C | I |
| Declare tenant segregation requirements | C | C | C | **A/R** |
| Grant/track exceptions | **A/R** | C | C | I |
| Pipeline-compromise incident response | **A/R** | R | C | I |

(R = Responsible, A = Accountable, C = Consulted, I = Informed.)

---

## 11. Conformance and assurance

- An item is **conformant** when all applicable baseline and tier MUST controls are implemented (Appendix A) and verifiable.
- Verification uses the appropriate evidence: automated policy evaluation and audit logs for technical controls; recorded attestation/review for process controls. Conformance is not assumed from log presence alone.
- Security Architecture MUST conduct periodic assurance reviews and report estate conformance under the weakest-link principle (§7.4).
- Non-conformance without an approved exception is a defect with a tier-based remediation timeline.

---

## 12. Exception management

- Any unmet MUST requires a formal exception: documented, risk-assessed, time-boxed, with a named owner, compensating control, and remediation date.
- Exceptions MUST be approved by Security Architecture and recorded centrally.
- Tier 3 MUST controls MAY be excepted only with senior risk-owner sign-off and a hard deadline.
- Open exceptions are reviewed at the standard's cadence; expired exceptions are escalated.
- SHOULD deviations need no exception but SHOULD be documented as engineering decisions.

---

## 13. Measuring effectiveness

Reported on two axes together so security and delivery are balanced.

**Delivery guardrails (DORA)** — MUST NOT be silently degraded: deployment frequency; lead time for changes; change-failure rate; time to restore service.

**Security posture metrics:**

| Metric | Type | Definition | Direction |
|---|---|---|---|
| Estate inventory coverage | Leading | % of platforms/repos under active discovery | → 100% |
| Baseline conformance rate | Leading | % of repos meeting all baseline MUSTs | → 100% |
| Drift detections open | Leading | Repos/pipelines below baseline | ↓ |
| Federated-identity coverage | Leading | % of platforms with SSO + federated lifecycle | → 100% |
| Standing-credential count | Leading | Long-lived prod/cloud creds held by pipelines | → 0 |
| Verified-deploy coverage | Leading | % of deployments with verified provenance | → 100% |
| Third-party-component governance | Leading | % of pipelines with allow-listed, pinned components | → 100% |
| Secret-leak MTTR | Lagging | Detection-to-rotation time | ↓ |
| Segregation findings (caught/escaped) | Lagging | Deploy-time segregation violations | caught ↑, escaped → 0 |
| Exception aging | Lagging | Open exceptions past remediation date | → 0 |
| Break-glass usage & review closure | Lagging | Emergency changes; % with completed reviews | reviews → 100% |

Targets MUST carry dates per tier; an open-ended "→ 100%" is a direction, not an objective.

---

## 14. Standard lifecycle

- **Versioning** — semantic; material control changes increment the version.
- **Ratification** — Security Architecture ratifies each version; only ratified versions are binding. This document is a draft until ratified.
- **Adoption** — on ratification, an adoption timeline by tier is published; new systems conform immediately, existing systems per the timeline.
- **Review** — at the stated cadence or on material estate/threat change.

---

## 15. Framework mappings

| Domain | OWASP Top 10 CI/CD | OWASP DevSecOps phase | DORA dimension |
|---|---|---|---|
| §6.1 Visibility | SEC-10 | Operate | MTTR |
| §6.2 Identity | SEC-2 | Develop/Operate | Change-fail rate |
| §6.3 Flow control | SEC-1 | Develop | Change-fail rate |
| §6.4 Execution integrity | SEC-4 | Build | Lead time |
| §6.5 Pipeline access | SEC-5 | Deploy | MTTR |
| §6.6 Secrets/keys | SEC-6 | Build/Deploy | Lead time |
| §6.7 Hardening | SEC-7 | Build | Change-fail rate |
| §6.8 Third-party/dependency | SEC-3, SEC-8 | Build/Test | Change-fail rate |
| §6.9 Artifact/provenance | SEC-9 | Release | Lead time |
| §6.10 Deploy/release | SEC-9 | Deploy | Deployment frequency, MTTR |
| §6.11 Tenant segregation | SEC-2, SEC-5, SEC-7 | Deploy/Operate | MTTR |
| §6.12 Logging/audit | SEC-10 | Operate | MTTR |
| §6.13 Resilience/emergency | — | Operate | MTTR |

---

## 16. References

- OWASP Top 10 CI/CD Security Risks (CICD-SEC-1 … 10) — https://owasp.org/www-project-top-10-ci-cd-security-risks/
- OWASP DevSecOps Guideline — https://owasp.org/www-project-devsecops-guideline/
- DORA (DevOps Research and Assessment) delivery metrics
- Recognised supply-chain integrity-level framework (for §6.9 / §9 provenance bars)

> If "DORA" is intended as the EU Digital Operational Resilience Act rather than the delivery metrics, §9, §11, and §13 are the hooks for a separate regulatory-mapping annex.

---

## Appendix A — Per-platform conformance register (template)

Maintained by platform owners; one column per in-scope platform. Status: ✅ native / 🟡 compensating + exception / ❌ gap.

| Control ref | Platform 1 (mechanism) | Status | Platform 2 (mechanism) | Status | Platform n … |
|---|---|---|---|---|---|
| 6.2.1 Federated SSO | | | | | |
| 6.2.2 Federated lifecycle/offboarding | | | | | |
| 6.3.1 Branch protection | | | | | |
| 6.3.3 Pre-merge secret scanning | | | | | |
| 6.4.1 Untrusted/trusted separation | | | | | |
| 6.6.2 No standing prod creds / federation | | | | | |
| 6.8.2 Third-party-component governance | | | | | |
| 6.9.1 Artifact signing & provenance | | | | | |
| 6.9.2 Verify at deploy/admission | | | | | |
| 6.10.3 Progressive multi-tenant deploy | | | | | |
| 6.11 Tenant segregation enforcement | | | | | |
| 6.12.1 Central audit streaming | | | | | |
| 6.13.2 Deployment freeze / kill switch | | | | | |

*(Extend rows to cover all applicable §6 requirements per platform.)*
