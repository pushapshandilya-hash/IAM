# CI/CD Security Standard

**Issued by:** Security Architecture  **Status:** Draft for review (not ratified)  **Version:** 0.7
Full metadata, lifecycle, and change history are in §9 (Document Control).

---

## 1. Purpose & Scope

**Purpose.** Define the mandatory security controls for all CI/CD pipelines across the organisation. These pipelines run on multiple, different source-control and CI/CD platforms and serve both internal teams/BUs and external customers/partners. Because security posture is set by the weakest-governed repository on the least-mature platform, the standard specifies control *intent* once and requires every platform to meet it.

**In scope.** All source-control platforms, CI/CD orchestrators, build/execution environments, artifact and secret stores, and deployment controllers used to deliver organisational software; all pipelines that produce a release-able artifact or effect a deployment; all human and non-human identities acting within them; and all tenants served.

**Out of scope** (governed elsewhere): developer endpoint security; application runtime security beyond what the pipeline deploys; production infrastructure not provisioned via CI/CD; corporate identity governance (a dependency, IAM domain); data classification (a dependency, TEN domain).

**Applicability.** Applies regardless of hosting model (self-hosted, managed, third-party-operated). Third-party-operated platforms MUST be brought into conformance contractually.

---

## 2. Definitions & Normative Language

**Normative keywords (RFC 2119).** **MUST/MUST NOT** = mandatory. **SHOULD/SHOULD NOT** = recommended; discretionary. **MAY** = optional. An unmet MUST requires a time-boxed exception (§7.4); a SHOULD does not.

**Terms.**
- **Provenance** — verifiable, signed metadata describing how and from what an artifact was built.
- **Control plane / data plane** — the CI/CD systems / the workloads they deploy.
- **Tenant** — an isolated context (code, config, secrets, data, runtime) that must be kept separate; *internal* (team/BU) or *external* (customer/partner).
- **Segregation model** — the declared mechanism keeping a tenant's resources separate.
- **Untrusted / trusted pipeline** — one operating on un-reviewed input / on reviewed, approved input with privileged downstream access.
- **Baseline** — the controls mandatory for any production pipeline; flagged **B** in §4 (**B†** where the target is shared/multi-tenant or external-facing).
- **Conformance register** — the record of how each platform implements each control (Appendix A).

---

## 3. Security Principles

**Zero Trust.** Assume any pipeline component can be compromised. Therefore: verify every actor and artifact explicitly (no implicit trust from network location or from a "green build"); grant least privilege, just-in-time, with no standing production credentials; and segment tenants and deployments to limit blast radius.

**Governing principles.**
- **Consistency** — one control intent applies to every platform; differences are implementation detail, not posture differences.
- **Differentiation** — control intensity scales with the risk tier of the repository/pipeline and tenant (§5).
- **Scalability** — controls are inherited by default; new repos/platforms/tenants are governed from day one.
- **Manageability** — Security Architecture sets central policy; platform owners implement; product teams self-serve within guardrails.

**Weakest-link reporting.** Overall security posture is reported at the level of its least-conformant component (§7.3).

---

## 4. Control Domains

~50 controls across 12 domains. Each control has a stable ID `CPS-<DOMAIN>-NN` (used in §5, §6, and Appendix A). **Level** follows §2. **Base** marks production-baseline controls: **B** always, **B†** when the target is shared/multi-tenant or external-facing.

### 4.1 Inventory & Visibility (VIS)
| ID | Control | Level | Base |
|---|---|---|---|
| CPS-VIS-01 | Maintain a continuous inventory of all platforms, orgs/projects, repositories, and pipelines; unknown repositories are non-conformant | MUST | B |
| CPS-VIS-02 | Detect newly created orgs/projects/repositories within a defined window | MUST | |
| CPS-VIS-03 | Tag each repository with owner and risk tier to drive automatic control application | SHOULD | |

### 4.2 Identity & Access (IAM)
| ID | Control | Level | Base |
|---|---|---|---|
| CPS-IAM-01 | Authenticate humans via central IdP SSO; no standalone/local accounts for standing access | MUST | B |
| CPS-IAM-02 | Federate joiner/mover/leaver so one deprovisioning removes access on all platforms | MUST | B |
| CPS-IAM-03 | Enforce least privilege via roles/groups with periodic recertification | MUST | B |
| CPS-IAM-04 | Inventory, own, scope, and lifecycle-manage non-human identities | MUST | |
| CPS-IAM-05 | Enforce segregation of duties: policy/exception authoring separated from production deploy | MUST | B |
| CPS-IAM-06 | Provision external collaborators with explicit scope and expiry; no default standing access | MUST | |
| CPS-IAM-07 | Scope each pipeline's privilege to its target resources/tenant; make it time-bound and revocable | MUST | B |

### 4.3 Source & Flow Control (SRC)
| ID | Control | Level | Base |
|---|---|---|---|
| CPS-SRC-01 | Protect production branches: no direct pushes; review by a non-author | MUST | B |
| CPS-SRC-02 | Keep pipeline definitions and policy-as-code in protected locations under stricter review | MUST | B |
| CPS-SRC-03 | Run secret scanning pre-merge; block on detection and trigger rotation | MUST | B |
| CPS-SRC-04 | Enforce commit/tag signing; keep unsigned changes out of production | SHOULD | |
| CPS-SRC-05 | Disable self-approval and check-bypass for production changes except via break-glass (CPS-RES-01) | MUST | B |

### 4.4 Pipeline Execution Integrity (EXE)
| ID | Control | Level | Base |
|---|---|---|---|
| CPS-EXE-01 | Separate untrusted from trusted pipelines so untrusted input cannot reach prod/cross-tenant credentials | MUST | B |
| CPS-EXE-02 | Run un-reviewed externally-triggered pipelines without secrets, production identity, or other tenants' resources | MUST | |
| CPS-EXE-03 | Isolate production build environments per job; no credential/state persistence enabling tampering or leakage (persistent environments are an exception) | MUST | B |
| CPS-EXE-04 | Prevent build caches/shared resources from crossing trust or tenant boundaries | MUST | B |

### 4.5 Secrets & Key Management (KEY)
| ID | Control | Level | Base |
|---|---|---|---|
| CPS-KEY-01 | Store secrets only in a managed store — never in source, pipeline definitions, logs, or artifacts | MUST | B |
| CPS-KEY-02 | Hold no standing long-lived production/cloud credentials; use short-lived federated workload identity | MUST | B |
| CPS-KEY-03 | Make secrets rotatable; automate rotation for high tiers | MUST | |
| CPS-KEY-04 | Scope secrets/keys so no single credential grants effective cross-tenant access | MUST | B |
| CPS-KEY-05 | Offer customer-managed keys where external-tenant tiers require them | SHOULD | |

### 4.6 Hardening & Configuration (CFG)
| ID | Control | Level | Base |
|---|---|---|---|
| CPS-CFG-01 | Configure each platform to a documented hardened baseline; make drift detectable | MUST | B |
| CPS-CFG-02 | Express security configuration as policy-as-code and enforce it automatically | MUST | B |

### 4.7 Dependencies & Third-Party Components (DEP)
| ID | Control | Level | Base |
|---|---|---|---|
| CPS-DEP-01 | Source dependencies/base components from approved, verified origins, pinned by version/digest | MUST | B |
| CPS-DEP-02 | Allow-list, pin, and review third-party pipeline components; none untrusted in trusted pipelines | MUST | B |
| CPS-DEP-03 | Produce an SBOM for every artifact | MUST | B |
| CPS-DEP-04 | Continuously monitor dependency/component risk against advisories | SHOULD | |

### 4.8 Artifact Integrity & Provenance (ART)
| ID | Control | Level | Base |
|---|---|---|---|
| CPS-ART-01 | Sign every release-able artifact with signed build provenance (source, inputs, builder identity) | MUST | B |
| CPS-ART-02 | Verify signatures and provenance at deploy/admission, not only at build | MUST | B |
| CPS-ART-03 | Make released versions immutable; never deploy mutable references to production | MUST | B |
| CPS-ART-04 | Make builds reproducible to support provenance verification | SHOULD | |

### 4.9 Deployment & Release (REL)
| ID | Control | Level | Base |
|---|---|---|---|
| CPS-REL-01 | Enforce tier-appropriate release-approval gates in the pipeline, not by convention | MUST | B |
| CPS-REL-02 | Deploy using short-lived, scoped, federated identity | MUST | B |
| CPS-REL-03 | Deploy progressively to multi-tenant targets with health gates that halt promotion on failure | MUST | B† |
| CPS-REL-04 | Provide automated, tested rollback | MUST | B |
| CPS-REL-05 | Validate target tenant scope and segregation before applying any change | MUST | B† |

### 4.10 Tenant Segregation (TEN)
| ID | Control | Level | Base |
|---|---|---|---|
| CPS-TEN-01 | Declare and enforce a segregation model for every tenant | MUST | B |
| CPS-TEN-02 | Deny cross-team/BU access by default (internal tenants) | MUST | B |
| CPS-TEN-03 | Segregate pipelines/credentials per external customer; no effective cross-customer access | MUST | B† |
| CPS-TEN-04 | Keep external-customer/regulated data out of shared/lower environments without de-identification | MUST | B† |

### 4.11 Logging, Audit & Detection (LOG)
| ID | Control | Level | Base |
|---|---|---|---|
| CPS-LOG-01 | Emit security-relevant audit events to a central append-only store with tier-based retention | MUST | B |
| CPS-LOG-02 | Normalise events across platforms for unified detection and investigation | MUST | |
| CPS-LOG-03 | Monitor and alert on privileged and anomalous control-plane actions | MUST | B |

### 4.12 Resilience & Emergency Change (RES)
| ID | Control | Level | Base |
|---|---|---|---|
| CPS-RES-01 | Maintain a logged, time-boxed break-glass procedure with extra authorisation and post-event review | MUST | B |
| CPS-RES-02 | Provide an organisation-wide/per-platform deployment freeze ("kill switch") to contain compromise | MUST | B |
| CPS-RES-03 | Keep pipeline configuration recoverable as code for disaster recovery | SHOULD | |

---

## 5. Risk Tiers

**Production baseline** = every control flagged **B** in §4 (plus **B†** where the target is shared/multi-tenant or external-facing). Tiers escalate requirements *above* baseline.

Each repository/pipeline and each tenant is assigned a tier; the **higher of the two governs**. External-facing and production-deploying repositories default higher.

| Area | Tier 1 (low / internal non-prod) | Tier 2 (standard prod) | Tier 3 (external / regulated / critical) |
|---|---|---|---|
| Review (CPS-SRC-01) | Non-author review | + designated owners | + ≥2 reviewers |
| Scanning gate (CPS-SRC-03 / DEP) | Warn | Block on high | Block on medium+ |
| Dynamic/interactive testing | Optional | Recommended | Required |
| Provenance (CPS-ART-*) | Basic signing | + verified at deploy | + higher integrity level, key ceremony |
| Secret rotation (CPS-KEY-03) | Manual | Automated | Automated + short TTL |
| Segregation assurance (CPS-TEN-*) | Logical | + validated at deploy | + independent review |
| Release approval (CPS-REL-01) | Automated gate | Automated gate | + manual approval at final stage |
| Key management (CPS-KEY-05) | Platform-managed | Platform-managed | Customer-managed option |
| Audit retention (CPS-LOG-01) | Standard | Extended | Compliance-defined |

---

## 6. Compliance Mapping

Control domains mapped to recognised catalogues for traceability: OWASP Top 10 CI/CD Security Risks, OWASP DevSecOps lifecycle phase, NIST SP 800-53 Rev 5 families, and CSA CCM v4 domains. (Delivery metrics are covered in §7.5.)

| Domain (IDs) | OWASP Top 10 CI/CD | DevSecOps phase | NIST 800-53 Rev 5 | CSA CCM v4 |
|---|---|---|---|---|
| VIS (01–03) | SEC-10 | Operate | CM-8, PM-5 | LOG / GRC |
| IAM (01–07) | SEC-2, SEC-5 | Develop / Operate | AC, IA | IAM |
| SRC (01–05) | SEC-1 | Develop | CM, SA-10, SA-15 | CCC |
| EXE (01–04) | SEC-4 | Build | SC, CM-7 | IVS / AIS |
| KEY (01–05) | SEC-6 | Build / Deploy | IA-5, SC-12, SC-28 | CEK |
| CFG (01–02) | SEC-7 | Build | CM-2, CM-6, CM-7 | CCC |
| DEP (01–04) | SEC-3, SEC-8 | Build / Test | SR, SA-12 | STA / AIS |
| ART (01–04) | SEC-9 | Release | SI-7, SR-4, SR-11 | CCC / AIS |
| REL (01–05) | SEC-9 | Deploy | CM-3, CM-4, SA-22 | CCC |
| TEN (01–04) | SEC-2, SEC-5, SEC-7 | Deploy / Operate | SC-2, SC-4, SC-7, AC-4 | IVS / DSP |
| LOG (01–03) | SEC-10 | Operate | AU, SI-4 | LOG |
| RES (01–03) | — | Operate | CP, IR | BCR / SEF |

---

## 7. Governance, Exceptions & Roles

### 7.1 Ownership
Security Architecture owns, issues, and maintains this standard; defines the baseline and tiers; ratifies versions; and approves exceptions. New repositories, pipelines, platforms, and tenants inherit the baseline by default — opting out requires an exception (§7.4); opting in requires no action.

### 7.2 Roles & responsibilities
| Activity | Security Architecture | Platform owner | Product/Eng team | Tenant owner |
|---|---|---|---|---|
| Own & ratify the standard | **A/R** | C | I | I |
| Define baseline & tiers | **A/R** | C | C | C |
| Implement controls on a platform | C | **A/R** | C | I |
| Maintain conformance register | C | **A/R** | I | I |
| Conform repos/pipelines | I | C | **A/R** | I |
| Inventory & drift detection | **A** | R | C | I |
| Declare tenant segregation needs | C | C | C | **A/R** |
| Grant/track exceptions | **A/R** | C | C | I |
| Pipeline-compromise response | **A/R** | R | C | I |

(R = Responsible, A = Accountable, C = Consulted, I = Informed.)

### 7.3 Cross-platform conformance & assurance
- Every applicable control MUST be implemented on every in-scope platform using that platform's capabilities; intent is uniform, mechanism may differ.
- Each platform owner MUST maintain a conformance-register entry per control (Appendix A): mechanism and status.
- A native gap MUST be recorded with a compensating control and an exception — never left silent.
- Automated posture/drift detection MUST run across all platforms; overall posture is reported under the weakest-link principle.
- Security Architecture MUST run periodic assurance reviews. Non-conformance without an approved exception is a defect with a tier-based remediation timeline.

### 7.4 Exceptions
- Any unmet MUST requires a formal, documented, risk-assessed, time-boxed exception with a named owner, compensating control, and remediation date, approved by Security Architecture and recorded centrally.
- Tier 3 MUST controls may be excepted only with senior risk-owner sign-off and a hard deadline.
- Expired exceptions are escalated. SHOULD deviations need no exception but SHOULD be documented.

### 7.5 Measurement
Reported with delivery metrics so security and speed stay balanced.
- **Delivery guardrails (DORA, must not be silently degraded):** deployment frequency; lead time for changes; change-failure rate; time to restore service.
- **Security posture metrics:** inventory coverage; baseline conformance rate; open drift detections; federated-identity coverage; standing-credential count (→0); verified-deploy coverage; third-party-component governance coverage; secret-leak MTTR; segregation findings (caught vs escaped); exception aging; break-glass usage and review-closure rate. Targets MUST carry dates per tier.

---

## 8. References

- OWASP Top 10 CI/CD Security Risks (CICD-SEC-1 … 10) — https://owasp.org/www-project-top-10-ci-cd-security-risks/
- OWASP DevSecOps Guideline — https://owasp.org/www-project-devsecops-guideline/
- NIST SP 800-53 Rev 5 — Security and Privacy Controls (control families referenced in §6)
- CSA Cloud Controls Matrix (CCM) v4 — cloud security control domains (referenced in §6)
- DORA (DevOps Research and Assessment) delivery metrics
- Recognised supply-chain integrity-level framework (for ART / Tier 3 provenance bars)

---

## 9. Document Control

| Field | Value |
|---|---|
| Title | CI/CD Security Standard |
| Control-ID namespace | `CPS-` (CI/CD Pipeline Security) |
| Issued by | Security Architecture |
| Status | Draft for review — not yet ratified |
| Version | 0.7 |
| Supersedes | v0.1–0.6 |
| Review cadence | Quarterly, or on material platform/threat change |

**Lifecycle.** Versioning is semantic; material control changes increment the version. Only versions ratified by Security Architecture are binding. On ratification, an adoption timeline by tier is published — new systems conform immediately, existing systems per the timeline.

**Change history.**
| Version | Summary |
|---|---|
| 0.7 | Adopted `CPS-<DOMAIN>-NN` control IDs (namespaced, zero-padded); folded the production baseline into §4 as a Base flag (single source of truth); replaced the §6 DORA column with a NIST 800-53 + CCM v4 crosswalk; trimmed definitions |
| 0.6 | Restructured into the concise 9-section form; controls converted to ID'd domain tables |
| 0.1–0.5 | Earlier drafts (initial scope, vendor-neutral rewrite, plain-language pass) |

**Note on "DORA".** Used here as the DevOps Research and Assessment delivery metrics. If the EU Digital Operational Resilience Act is also meant, §5, §7.3, and §7.5 are the hooks for a separate regulatory-mapping annex.

---

## Appendix A — Per-platform conformance register (template)

Maintained by platform owners; one column per in-scope platform. Status: ✅ native / 🟡 compensating + exception / ❌ gap. Extend rows to cover all applicable controls.

| Control ID | Platform 1 (mechanism) | Status | Platform 2 (mechanism) | Status | Platform n … |
|---|---|---|---|---|---|
| CPS-IAM-01 | | | | | |
| CPS-IAM-02 | | | | | |
| CPS-SRC-01 | | | | | |
| CPS-SRC-03 | | | | | |
| CPS-EXE-01 | | | | | |
| CPS-KEY-02 | | | | | |
| CPS-DEP-02 | | | | | |
| CPS-ART-01 | | | | | |
| CPS-ART-02 | | | | | |
| CPS-REL-03 | | | | | |
| CPS-TEN-01 | | | | | |
| CPS-LOG-01 | | | | | |
| CPS-RES-02 | | | | | |
