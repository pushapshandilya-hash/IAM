# CI/CD Security Standard

**Issued by:** Security Architecture  **Status:** Draft for review (not ratified)  **Version:** 0.6
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
- **Pipeline** — an automated sequence from input (commit, trigger) to artifact and/or deployment.
- **Artifact** — any release-able build output.
- **Provenance** — verifiable, signed metadata describing how and from what an artifact was built.
- **Control plane / data plane** — the CI/CD systems / the workloads they deploy.
- **Tenant** — an isolated context (code, config, secrets, data, runtime) that must be kept separate; *internal* (team/BU) or *external* (customer/partner).
- **Segregation model** — the declared mechanism keeping a tenant's resources separate.
- **Untrusted / trusted pipeline** — one operating on un-reviewed input / on reviewed, approved input with privileged downstream access.
- **Baseline** — the controls mandatory for any production pipeline (§5).
- **Conformance register** — the record of how each platform implements each control (Appendix A).

---

## 3. Security Principles

**Zero Trust.** Assume any pipeline component can be compromised. Therefore: verify every actor and artifact explicitly (no implicit trust from network location or from a "green build"); grant least privilege, just-in-time, with no standing production credentials; and segment tenants and deployments to limit blast radius.

**Governing principles.**
- **Consistency** — one control intent applies to every platform; differences are implementation detail, not posture differences.
- **Differentiation** — control intensity scales with the risk tier of the repository/pipeline and tenant (§5.2).
- **Scalability** — controls are inherited by default; new repos/platforms/tenants are governed from day one.
- **Manageability** — Security Architecture sets central policy; platform owners implement; product teams self-serve within guardrails.

**Weakest-link reporting.** Overall security posture is reported at the level of its least-conformant component (§7.3).

---

## 4. Control Domains

~50 controls across 12 domains. Each control has a stable ID (used in §5 and §6). Levels follow §2.

### 4.1 Inventory & Visibility (VIS)
| ID | Control | Level |
|---|---|---|
| VIS-1 | Maintain a continuous inventory of all platforms, orgs/projects, repositories, and pipelines; unknown repositories are non-conformant | MUST |
| VIS-2 | Detect newly created orgs/projects/repositories within a defined window | MUST |
| VIS-3 | Tag each repository with owner and risk tier to drive automatic control application | SHOULD |

### 4.2 Identity & Access (IAM)
| ID | Control | Level |
|---|---|---|
| IAM-1 | Authenticate humans via central IdP SSO; no standalone/local accounts for standing access | MUST |
| IAM-2 | Federate joiner/mover/leaver so one deprovisioning removes access on all platforms | MUST |
| IAM-3 | Enforce least privilege via roles/groups with periodic recertification | MUST |
| IAM-4 | Inventory, own, scope, and lifecycle-manage non-human identities | MUST |
| IAM-5 | Enforce segregation of duties: policy/exception authoring separated from production deploy | MUST |
| IAM-6 | Provision external collaborators with explicit scope and expiry; no default standing access | MUST |
| IAM-7 | Scope each pipeline's privilege to its target resources/tenant; make it time-bound and revocable | MUST |

### 4.3 Source & Flow Control (SRC)
| ID | Control | Level |
|---|---|---|
| SRC-1 | Protect production branches: no direct pushes; review by a non-author | MUST |
| SRC-2 | Keep pipeline definitions and policy-as-code in protected locations under stricter review | MUST |
| SRC-3 | Run secret scanning pre-merge; block on detection and trigger rotation | MUST |
| SRC-4 | Enforce commit/tag signing; keep unsigned changes out of production | SHOULD |
| SRC-5 | Disable self-approval and check-bypass for production changes except via break-glass (RES-1) | MUST |

### 4.4 Pipeline Execution Integrity (EXE)
| ID | Control | Level |
|---|---|---|
| EXE-1 | Separate untrusted from trusted pipelines so untrusted input cannot reach prod/cross-tenant credentials | MUST |
| EXE-2 | Run un-reviewed externally-triggered pipelines without secrets, production identity, or other tenants' resources | MUST |
| EXE-3 | Isolate production build environments per job; no credential/state persistence enabling tampering or leakage (persistent environments are an exception) | MUST |
| EXE-4 | Prevent build caches/shared resources from crossing trust or tenant boundaries | MUST |

### 4.5 Secrets & Key Management (KEY)
| ID | Control | Level |
|---|---|---|
| KEY-1 | Store secrets only in a managed store — never in source, pipeline definitions, logs, or artifacts | MUST |
| KEY-2 | Hold no standing long-lived production/cloud credentials; use short-lived federated workload identity | MUST |
| KEY-3 | Make secrets rotatable; automate rotation for high tiers | MUST |
| KEY-4 | Scope secrets/keys so no single credential grants effective cross-tenant access | MUST |
| KEY-5 | Offer customer-managed keys where external-tenant tiers require them | SHOULD |

### 4.6 Hardening & Configuration (CFG)
| ID | Control | Level |
|---|---|---|
| CFG-1 | Configure each platform to a documented hardened baseline; make drift detectable | MUST |
| CFG-2 | Express security configuration as policy-as-code and enforce it automatically | MUST |

### 4.7 Dependencies & Third-Party Components (DEP)
| ID | Control | Level |
|---|---|---|
| DEP-1 | Source dependencies/base components from approved, verified origins, pinned by version/digest | MUST |
| DEP-2 | Allow-list, pin, and review third-party pipeline components; none untrusted in trusted pipelines | MUST |
| DEP-3 | Produce an SBOM for every artifact | MUST |
| DEP-4 | Continuously monitor dependency/component risk against advisories | SHOULD |

### 4.8 Artifact Integrity & Provenance (ART)
| ID | Control | Level |
|---|---|---|
| ART-1 | Sign every release-able artifact with signed build provenance (source, inputs, builder identity) | MUST |
| ART-2 | Verify signatures and provenance at deploy/admission, not only at build | MUST |
| ART-3 | Make released versions immutable; never deploy mutable references to production | MUST |
| ART-4 | Make builds reproducible to support provenance verification | SHOULD |

### 4.9 Deployment & Release (REL)
| ID | Control | Level |
|---|---|---|
| REL-1 | Enforce tier-appropriate release-approval gates in the pipeline, not by convention | MUST |
| REL-2 | Deploy using short-lived, scoped, federated identity | MUST |
| REL-3 | Deploy progressively to multi-tenant targets with health gates that halt promotion on failure | MUST |
| REL-4 | Provide automated, tested rollback | MUST |
| REL-5 | Validate target tenant scope and segregation before applying any change | MUST |

### 4.10 Tenant Segregation (TEN)
| ID | Control | Level |
|---|---|---|
| TEN-1 | Declare and enforce a segregation model for every tenant | MUST |
| TEN-2 | Deny cross-team/BU access by default (internal tenants) | MUST |
| TEN-3 | Segregate pipelines/credentials per external customer; no effective cross-customer access | MUST |
| TEN-4 | Keep external-customer/regulated data out of shared/lower environments without de-identification | MUST |

### 4.11 Logging, Audit & Detection (LOG)
| ID | Control | Level |
|---|---|---|
| LOG-1 | Emit security-relevant audit events to a central append-only store with tier-based retention | MUST |
| LOG-2 | Normalise events across platforms for unified detection and investigation | MUST |
| LOG-3 | Monitor and alert on privileged and anomalous control-plane actions | MUST |

### 4.12 Resilience & Emergency Change (RES)
| ID | Control | Level |
|---|---|---|
| RES-1 | Maintain a logged, time-boxed break-glass procedure with extra authorisation and post-event review | MUST |
| RES-2 | Provide an organisation-wide/per-platform deployment freeze ("kill switch") to contain compromise | MUST |
| RES-3 | Keep pipeline configuration recoverable as code for disaster recovery | SHOULD |

---

## 5. Minimum Production Baseline

### 5.1 Baseline controls
Every pipeline that deploys to production MUST meet all of the following, regardless of tier. (Controls marked † apply when the target is shared/multi-tenant or external-customer-facing.)

| Control | Baseline requirement |
|---|---|
| VIS-1 | Repository is in the inventory and owned |
| IAM-1 | SSO via central IdP; no local accounts |
| IAM-2 | Federated joiner/mover/leaver |
| IAM-3 | Least-privilege, recertified access |
| IAM-5 | Segregation of duties enforced |
| IAM-7 | Pipeline privilege scoped, time-bound, revocable |
| SRC-1 | Protected branches with non-author review |
| SRC-2 | Protected pipeline definitions / policy-as-code |
| SRC-3 | Pre-merge secret scanning + rotation |
| SRC-5 | No self-approval / no check bypass (except break-glass) |
| EXE-1 | Untrusted/trusted pipeline separation |
| EXE-3 | Per-job isolated production build environments |
| EXE-4 | No cross-boundary caches/shared resources |
| KEY-1 | Secrets only in a managed store |
| KEY-2 | No standing prod/cloud credentials; federated identity |
| KEY-4 | No single credential grants cross-tenant access |
| CFG-1 | Hardened platform baseline with drift detection |
| CFG-2 | Security config as enforced policy-as-code |
| DEP-1 | Approved, pinned dependencies |
| DEP-2 | Governed third-party pipeline components |
| DEP-3 | SBOM per artifact |
| ART-1 | Signed artifact + provenance |
| ART-2 | Signature/provenance verified at deploy |
| ART-3 | Immutable released versions |
| REL-1 | Pipeline-enforced release approval |
| REL-2 | Short-lived federated deploy identity |
| REL-3 † | Progressive deploy with health gates |
| REL-4 | Automated, tested rollback |
| REL-5 † | Pre-apply tenant-scope/segregation validation |
| TEN-1 | Declared, enforced segregation model |
| TEN-3 † | Per-external-customer segregation |
| TEN-4 † | No customer/regulated data in lower environments |
| LOG-1 | Central append-only audit logging |
| LOG-3 | Privileged/anomalous action monitoring |
| RES-1 | Break-glass procedure |
| RES-2 | Deployment freeze capability |

### 5.2 Tier escalation above baseline
Each repository/pipeline and each tenant is assigned a tier; the **higher of the two governs**. External-facing and production-deploying repositories default higher.

| Area | Tier 1 (low / internal non-prod) | Tier 2 (standard prod) | Tier 3 (external / regulated / critical) |
|---|---|---|---|
| Review (SRC-1) | Non-author review | + designated owners | + ≥2 reviewers |
| Scanning gate (SRC-3/DEP) | Warn | Block on high | Block on medium+ |
| Dynamic/interactive testing | Optional | Recommended | Required |
| Provenance (ART) | Basic signing | + verified at deploy | + higher integrity level, key ceremony |
| Secret rotation (KEY-3) | Manual | Automated | Automated + short TTL |
| Segregation assurance (TEN) | Logical | + validated at deploy | + independent review |
| Release approval (REL-1) | Automated gate | Automated gate | + manual approval at final stage |
| Key management (KEY-5) | Platform-managed | Platform-managed | Customer-managed option |
| Audit retention (LOG-1) | Standard | Extended | Compliance-defined |

---

## 6. Compliance Mapping

Control domains mapped to the OWASP Top 10 CI/CD Security Risks, the OWASP DevSecOps lifecycle phase, and the DORA delivery dimension most affected.

| Domain (control IDs) | OWASP Top 10 CI/CD | DevSecOps phase | DORA |
|---|---|---|---|
| VIS (1–3) | SEC-10 | Operate | Time to restore |
| IAM (1–7) | SEC-2, SEC-5 | Develop / Operate | Change-fail rate |
| SRC (1–5) | SEC-1 | Develop | Change-fail rate |
| EXE (1–4) | SEC-4 | Build | Lead time |
| KEY (1–5) | SEC-6 | Build / Deploy | Lead time |
| CFG (1–2) | SEC-7 | Build | Change-fail rate |
| DEP (1–4) | SEC-3, SEC-8 | Build / Test | Change-fail rate |
| ART (1–4) | SEC-9 | Release | Lead time |
| REL (1–5) | SEC-9 | Deploy | Deployment frequency, Time to restore |
| TEN (1–4) | SEC-2, SEC-5, SEC-7 | Deploy / Operate | Time to restore |
| LOG (1–3) | SEC-10 | Operate | Time to restore |
| RES (1–3) | — | Operate | Time to restore |

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
- DORA (DevOps Research and Assessment) delivery metrics
- Recognised supply-chain integrity-level framework (for ART / Tier 3 provenance bars)

---

## 9. Document Control

| Field | Value |
|---|---|
| Title | CI/CD Security Standard |
| Issued by | Security Architecture |
| Status | Draft for review — not yet ratified |
| Version | 0.6 |
| Supersedes | v0.1–0.5 |
| Review cadence | Quarterly, or on material platform/threat change |

**Lifecycle.** Versioning is semantic; material control changes increment the version. Only versions ratified by Security Architecture are binding. On ratification, an adoption timeline by tier is published — new systems conform immediately, existing systems per the timeline.

**Change history.**
| Version | Summary |
|---|---|
| 0.6 | Restructured into concise 9-section form; controls converted to ID'd domain tables; added consolidated production baseline; merged governance/roles/exceptions/assurance |
| 0.1–0.5 | Earlier drafts (initial scope, vendor-neutral rewrite, plain-language pass) |

**Note on "DORA".** Used here as the DevOps Research and Assessment delivery metrics. If the EU Digital Operational Resilience Act is also meant, §5.2, §7.3, and §7.5 are the hooks for a separate regulatory-mapping annex.

---

## Appendix A — Per-platform conformance register (template)

Maintained by platform owners; one column per in-scope platform. Status: ✅ native / 🟡 compensating + exception / ❌ gap. Extend rows to cover all applicable controls.

| Control ID | Platform 1 (mechanism) | Status | Platform 2 (mechanism) | Status | Platform n … |
|---|---|---|---|---|---|
| IAM-1 | | | | | |
| IAM-2 | | | | | |
| SRC-1 | | | | | |
| SRC-3 | | | | | |
| EXE-1 | | | | | |
| KEY-2 | | | | | |
| DEP-2 | | | | | |
| ART-1 | | | | | |
| ART-2 | | | | | |
| REL-3 | | | | | |
| TEN-1 | | | | | |
| LOG-1 | | | | | |
| RES-2 | | | | | |
