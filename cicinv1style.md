# CI/CD Security Standard

**Issued by:** Security Architecture  **Status:** Draft for review (not ratified)  **Version:** 0.9
Full metadata, lifecycle, and change history are in §10 (Document Control).

This standard defines the security controls that every CI/CD pipeline in the organisation must meet. Each control is given with a short statement of **what it is** and **why it is needed**, so the requirement and its intent travel together. Sections 1–3 set context; §4 carries the general controls; §5 adds controls specific to pipelines that build or deploy AI/ML systems; §6–§10 cover tiering, mapping, governance, references, and document control.

---

## 1. Purpose & Scope

**Purpose.** Define the mandatory security controls for all CI/CD pipelines across the organisation. These pipelines run on multiple, different source-control and CI/CD platforms and serve both internal teams/BUs and external customers/partners. Because security posture is set by the weakest-governed repository on the least-mature platform, the standard specifies control *intent* once and requires every platform to meet it.

**In scope.** All source-control platforms, CI/CD orchestrators, build/execution environments, artifact and secret stores, and deployment controllers used to deliver organisational software; all pipelines that produce a release-able artifact or effect a deployment — **including pipelines that train, build, or deploy AI/ML models** (§5); all human and non-human identities acting within them; and all tenants served.

**Out of scope** (governed elsewhere): developer endpoint security; application/model runtime security beyond what the pipeline deploys; production infrastructure not provisioned via CI/CD; corporate identity governance (a dependency, IAM domain); data classification (a dependency, TEN domain).

**Applicability.** Applies regardless of hosting model (self-hosted, managed, third-party-operated). Third-party-operated platforms MUST be brought into conformance contractually.

---

## 2. Definitions & Normative Language

**Normative keywords (RFC 2119).** **MUST/MUST NOT** = mandatory. **SHOULD/SHOULD NOT** = recommended; discretionary. **MAY** = optional. An unmet MUST requires a time-boxed exception (§8.4); a SHOULD does not.

**Terms.**
- **Provenance** — verifiable, signed metadata describing how and from what an artifact was built.
- **Control plane / data plane** — the CI/CD systems / the workloads they deploy.
- **Tenant** — an isolated context (code, config, secrets, data, runtime) that must be kept separate; *internal* (team/BU) or *external* (customer/partner).
- **Segregation model** — the declared mechanism keeping a tenant's resources separate.
- **Untrusted / trusted pipeline** — one operating on un-reviewed input / on reviewed, approved input with privileged downstream access.
- **Model artifact** — a trained model plus the metadata needed to trust and reproduce it (training/eval data references, code commit, hyperparameters, evaluation results, model card).
- **Baseline** — the controls mandatory for any production pipeline; flagged **B** in §4–§5 (**B†** where the target is shared/multi-tenant or external-facing).
- **Conformance register** — the record of how each platform implements each control (Appendix A).

---

## 3. Threat Model & Principles

**Why CI/CD is a target.** Pipelines are the shortest path from a commit to production and usually run with high-privileged, standing credentials, so a single pipeline compromise can poison every downstream artifact and — in a multi-tenant estate — reach every tenant at once. Public breaches such as SolarWinds and Codecov follow the same pattern: attackers compromise the factory, not the product. The controls in §4–§5 exist to mitigate the OWASP Top 10 CI/CD Security Risks (mapped in §7).

**Posture.** This standard assumes breach and applies Zero Trust as its lens: verify every actor and artifact explicitly (no implicit trust from network location or a "green build"), grant least privilege just-in-time with no standing production credentials, and segment tenants and deployments to limit blast radius. These tenets are realised *through the controls themselves* (notably IAM, KEY, EXE, TEN), not as a separate control set; for the architectural model see the OWASP Zero Trust Architecture cheat sheet (§9).

**Governing principles.**
- **Consistency** — one control intent applies to every platform; differences are implementation detail, not posture differences.
- **Differentiation** — control intensity scales with the risk tier of the repository/pipeline and tenant (§6).
- **Scalability** — controls are inherited by default; new repos/platforms/tenants are governed from day one.
- **Manageability** — Security Architecture sets central policy; platform owners implement; product teams self-serve within guardrails.

**Weakest-link reporting.** Overall security posture is reported at the level of its least-conformant component (§8.3).

---

## 4. Control Domains

~50 controls across 12 domains. Each control has a stable ID `CPS-<DOMAIN>-NN` (used in §6, §7, and Appendix A). **Level** follows §2. **Base** marks production-baseline controls: **B** always, **B†** when the target is shared/multi-tenant or external-facing. Each domain opens with a summary table, then per-control detail.

### 4.1 Inventory & Visibility (VIS)
*You cannot secure what you cannot see. This domain ensures every platform, repository, and pipeline is known and owned — the precondition for every other control.*

| ID | Control | Level | Base |
|---|---|---|---|
| CPS-VIS-01 | Continuous inventory of all platforms, orgs/projects, repositories, and pipelines | MUST | B |
| CPS-VIS-02 | Detect newly created orgs/projects/repositories within a defined window | MUST | |
| CPS-VIS-03 | Tag each repository with owner and risk tier | SHOULD | |

**CPS-VIS-01 — Continuous inventory** · `MUST` · Baseline
- *What it is.* A continuously maintained, authoritative inventory of every platform, org/project, repository, and pipeline; anything not in it is treated as non-conformant.
- *Why it's needed.* Coverage is the prerequisite for every other control — an unknown repository is an ungoverned one, and ungoverned repositories are where drift, shadow IT, and attackers live.

**CPS-VIS-02 — Detect new assets** · `MUST`
- *What it is.* Newly created orgs, projects, or repositories are discovered within a defined detection window.
- *Why it's needed.* Closes the gap in which a freshly created (or shadow) repository could operate ungoverned between creation and discovery.

**CPS-VIS-03 — Ownership & tier metadata** · `SHOULD`
- *What it is.* Each repository carries owner and risk-tier metadata.
- *Why it's needed.* Lets controls and tiers be applied automatically at scale instead of by manual triage.

### 4.2 Identity & Access (IAM)
*Controls who and what can act in the pipeline, across every platform, under least privilege and a single identity source.*

| ID | Control | Level | Base |
|---|---|---|---|
| CPS-IAM-01 | Central IdP SSO; no standalone accounts for standing access | MUST | B |
| CPS-IAM-02 | Federated joiner/mover/leaver across all platforms | MUST | B |
| CPS-IAM-03 | Least privilege via roles/groups, periodically recertified | MUST | B |
| CPS-IAM-04 | Inventory, own, scope, lifecycle-manage non-human identities | MUST | |
| CPS-IAM-05 | Segregation of duties: policy/exception authoring vs production deploy | MUST | B |
| CPS-IAM-06 | Governed external collaborators with scope and expiry | MUST | |
| CPS-IAM-07 | Pipeline privilege scoped to target, time-bound, revocable | MUST | B |
| CPS-IAM-08 | Multi-factor authentication for human and privileged access | MUST | B |

**CPS-IAM-01 — Central SSO** · `MUST` · Baseline
- *What it is.* All human access to every platform authenticates through the central identity provider via SSO; standalone/local accounts are not used for standing access.
- *Why it's needed.* One authoritative identity source makes policy consistent and revocation instant; local accounts evade both and survive offboarding.

**CPS-IAM-02 — Federated lifecycle** · `MUST` · Baseline
- *What it is.* Joiner/mover/leaver changes propagate so one deprovisioning action removes access on all platforms.
- *Why it's needed.* In a multi-platform estate, leavers routinely retain access on a forgotten platform — a leading insider and credential-theft risk.

**CPS-IAM-03 — Least privilege** · `MUST` · Baseline
- *What it is.* Access is least-privilege, granted to roles/groups, and recertified periodically.
- *Why it's needed.* Limits the blast radius of any compromised account and catches privilege creep before it accumulates.

**CPS-IAM-04 — Non-human identities** · `MUST`
- *What it is.* Service accounts and automation identities are inventoried, owned, scoped, and lifecycle-managed like human ones.
- *Why it's needed.* Non-human identities outnumber humans, are frequently unowned and over-privileged, and are a prime target for persistence.

**CPS-IAM-05 — Segregation of duties** · `MUST` · Baseline
- *What it is.* Authoring security policy or granting exceptions is separated from the ability to deploy to production.
- *Why it's needed.* Prevents one actor from both weakening a control and exploiting the gap they created.

**CPS-IAM-06 — External collaborators** · `MUST`
- *What it is.* External contributors are provisioned through a governed process with explicit scope and expiry; no standing access by default.
- *Why it's needed.* External collaborators are a common entry point; bounded, expiring access caps the exposure.

**CPS-IAM-07 — Pipeline privilege scope** · `MUST` · Baseline
- *What it is.* Each pipeline's privilege is scoped to its target resources/tenant and is time-bound and revocable.
- *Why it's needed.* An over-scoped pipeline is a high-value target whose compromise reaches far beyond its own job.

**CPS-IAM-08 — Multi-factor authentication** · `MUST` · Baseline
- *What it is.* Human access to every platform, and all privileged or production-affecting actions, require multi-factor authentication.
- *Why it's needed.* Passwords and tokens are routinely phished or leaked; MFA blocks the most common account-takeover path into the pipeline.

### 4.3 Source & Flow Control (SRC)
*Protects the integrity of what enters the pipeline — code, pipeline definitions, and configuration.*

| ID | Control | Level | Base |
|---|---|---|---|
| CPS-SRC-01 | Protected production branches; non-author review | MUST | B |
| CPS-SRC-02 | Protected pipeline definitions / policy-as-code, stricter review | MUST | B |
| CPS-SRC-03 | Pre-merge secret scanning; block + rotate on detection | MUST | B |
| CPS-SRC-04 | Commit/tag signing; unsigned changes kept out of production | SHOULD | |
| CPS-SRC-05 | No self-approval / no check bypass except via break-glass | MUST | B |

**CPS-SRC-01 — Branch protection** · `MUST` · Baseline
- *What it is.* Production-bound branches block direct pushes and require review by someone other than the author. Auto-merge that could bypass review is disabled, forking of private/internal repositories is restricted, and the ability to change a repository's visibility to public is controlled.
- *Why it's needed.* Enforces four-eyes and stops unreviewed change reaching production; auto-merge, uncontrolled forks, and accidental public exposure are common ways the review boundary is silently bypassed.

**CPS-SRC-02 — Protected pipeline definitions** · `MUST` · Baseline
- *What it is.* Pipeline definitions and policy-as-code live in protected locations under stricter review than application code.
- *Why it's needed.* Whoever can edit the pipeline can subvert every control it enforces — the classic Poisoned Pipeline Execution path.

**CPS-SRC-03 — Pre-merge secret scanning** · `MUST` · Baseline
- *What it is.* Secret scanning runs before merge; a hit blocks the merge and triggers rotation.
- *Why it's needed.* Catches secrets before they enter history, where they are hard to purge and trivial to harvest.

**CPS-SRC-04 — Commit/tag signing** · `SHOULD`
- *What it is.* Commits and tags are signed and verified; unsigned changes are kept out of production.
- *Why it's needed.* Establishes authorship integrity and blocks spoofed or forged commits.

**CPS-SRC-05 — No self-approval or bypass** · `MUST` · Baseline
- *What it is.* Self-approval and check-bypass are disabled for production changes except through break-glass (CPS-RES-01).
- *Why it's needed.* Bypass paths are where controls quietly fail; removing them prevents silent circumvention.

### 4.4 Pipeline Execution Integrity (EXE)
*Protects the build/execution step itself from being hijacked or used to cross a trust boundary.*

| ID | Control | Level | Base |
|---|---|---|---|
| CPS-EXE-01 | Separate untrusted from trusted pipelines | MUST | B |
| CPS-EXE-02 | External-triggered un-reviewed pipelines run without secrets/prod identity/other tenants | MUST | |
| CPS-EXE-03 | Per-job isolated production build environments | MUST | B |
| CPS-EXE-04 | No build cache/shared resource across trust or tenant boundaries | MUST | B |

**CPS-EXE-01 — Untrusted/trusted separation** · `MUST` · Baseline
- *What it is.* Pipelines running un-reviewed input are architecturally separated from those holding production or cross-tenant credentials.
- *Why it's needed.* Primary defence against Poisoned Pipeline Execution — it removes the path from a malicious contribution to production privilege.

**CPS-EXE-02 — External-trigger isolation** · `MUST`
- *What it is.* Pipelines triggered by un-reviewed external input run without secrets, production identity, or any other tenant's resources.
- *Why it's needed.* A malicious pull request must not be able to execute privileged steps or read secrets.

**CPS-EXE-03 — Per-job isolation** · `MUST` · Baseline
- *What it is.* Production builds run on isolated nodes, per job, with no privileged or host-level access (e.g., no privileged containers) and no credentials/state persisted across jobs in a way that enables tampering or leakage; persistent environments are a declared exception with compensating isolation.
- *Why it's needed.* Shared, persistent, or over-privileged runners enable cache poisoning, container escape, and lateral movement between jobs and tenants.

**CPS-EXE-04 — No cross-boundary cache** · `MUST` · Baseline
- *What it is.* Build caches and shared resources do not cross trust or tenant boundaries.
- *Why it's needed.* A poisoned shared cache silently contaminates downstream builds or leaks one tenant's data into another's.

### 4.5 Secrets & Key Management (KEY)
*Keeps credentials and keys out of reach and prevents standing or cross-tenant access.*

| ID | Control | Level | Base |
|---|---|---|---|
| CPS-KEY-01 | Secrets only in a managed store | MUST | B |
| CPS-KEY-02 | No standing prod/cloud credentials; federated workload identity | MUST | B |
| CPS-KEY-03 | Rotatable secrets; automated rotation at high tiers | MUST | |
| CPS-KEY-04 | No single credential grants effective cross-tenant access | MUST | B |
| CPS-KEY-05 | Customer-managed keys where external tiers require | SHOULD | |

**CPS-KEY-01 — Managed secret store** · `MUST` · Baseline
- *What it is.* Secrets are held only in a managed secret store — never in source, pipeline definitions, logs, or artifacts.
- *Why it's needed.* Secrets in source/logs/artifacts are the most common and most exploited leakage path.

**CPS-KEY-02 — No standing credentials** · `MUST` · Baseline
- *What it is.* No platform holds standing long-lived production/cloud credentials; short-lived federated workload identity is used instead.
- *Why it's needed.* Standing credentials are the prize in a pipeline breach; federation removes the durable target.

**CPS-KEY-03 — Rotation** · `MUST`
- *What it is.* Secrets are rotatable, with rotation automated for high tiers.
- *Why it's needed.* Limits the useful lifetime of any leaked secret.

**CPS-KEY-04 — Cross-tenant scoping** · `MUST` · Baseline
- *What it is.* Secrets and keys are scoped so no single credential grants effective cross-tenant access (hierarchical key schemes are permitted if cross-tenant access is not achievable).
- *Why it's needed.* Prevents one compromised credential from becoming a multi-tenant breach.

**CPS-KEY-05 — Customer-managed keys** · `SHOULD`
- *What it is.* Customer-managed key options are offered where external-tenant tiers require them.
- *Why it's needed.* Meets contractual and regulatory key-control obligations for external customers.

### 4.6 Hardening & Configuration (CFG)
*Keeps each platform configured to a known-good, drift-detected baseline.*

| ID | Control | Level | Base |
|---|---|---|---|
| CPS-CFG-01 | Documented hardened baseline per platform; drift detectable | MUST | B |
| CPS-CFG-02 | Security configuration as enforced policy-as-code | MUST | B |
| CPS-CFG-03 | Secure transport (TLS) between platforms, integrations, and runners | MUST | B |

**CPS-CFG-01 — Hardened baseline** · `MUST` · Baseline
- *What it is.* Each platform is configured to a documented hardened baseline, and drift from it is detectable.
- *Why it's needed.* Default platform configurations are permissive, and drift quietly reopens risks that were previously closed.

**CPS-CFG-02 — Policy-as-code config** · `MUST` · Baseline
- *What it is.* Security-relevant configuration is expressed as policy-as-code and enforced automatically.
- *Why it's needed.* Convention-based configuration is unenforceable and unauditable at scale.

**CPS-CFG-03 — Secure transport** · `MUST` · Baseline
- *What it is.* Communication between source-control, CI/CD orchestrators, runners, and integrations uses current, strong transport encryption (TLS 1.2+ or better).
- *Why it's needed.* Unencrypted or weak inter-platform links expose credentials, source, and artifacts to interception and tampering in transit — a real risk across a multi-platform estate where systems talk to each other constantly.

### 4.7 Dependencies & Third-Party Components (DEP)
*Governs everything the pipeline pulls in from outside — dependencies and third-party pipeline components.*

| ID | Control | Level | Base |
|---|---|---|---|
| CPS-DEP-01 | Approved, verified, pinned dependencies/base components | MUST | B |
| CPS-DEP-02 | Allow-listed, pinned, reviewed third-party pipeline components | MUST | B |
| CPS-DEP-03 | SBOM for every artifact | MUST | B |
| CPS-DEP-04 | Continuous dependency/component risk monitoring | SHOULD | |

**CPS-DEP-01 — Pinned dependencies** · `MUST` · Baseline
- *What it is.* Dependencies and base components come from approved, integrity-verified origins, pinned by version/digest.
- *Why it's needed.* Defeats dependency-confusion and tampered-package attacks that exploit unpinned or unverified sources.

**CPS-DEP-02 — Third-party pipeline components** · `MUST` · Baseline
- *What it is.* Third-party build steps, plugins, extensions, and reusable workflows are allow-listed by source, pinned to an immutable reference, and reviewed; none untrusted run in trusted pipelines.
- *Why it's needed.* Third-party pipeline components execute with pipeline privilege — a direct Poisoned Pipeline Execution and supply-chain vector.

**CPS-DEP-03 — SBOM** · `MUST` · Baseline
- *What it is.* A software bill of materials is produced for every artifact.
- *Why it's needed.* Without a component inventory you cannot answer "are we affected?" when a new advisory lands.

**CPS-DEP-04 — Advisory monitoring** · `SHOULD`
- *What it is.* Dependency and component risk is monitored continuously against new advisories.
- *Why it's needed.* Today's clean dependency is tomorrow's CVE; standing monitoring catches the change.

### 4.8 Artifact Integrity & Provenance (ART)
*Ensures what you ship is what you built and approved, through signing and provenance.*

| ID | Control | Level | Base |
|---|---|---|---|
| CPS-ART-01 | Signed artifacts with signed build provenance | MUST | B |
| CPS-ART-02 | Verify signatures/provenance at deploy/admission | MUST | B |
| CPS-ART-03 | Immutable released versions; no mutable refs to production | MUST | B |
| CPS-ART-04 | Reproducible builds | SHOULD | |

**CPS-ART-01 — Sign + provenance** · `MUST` · Baseline
- *What it is.* Every release-able artifact is signed with signed build provenance (source, inputs, builder identity).
- *Why it's needed.* Lets consumers verify origin and integrity rather than trusting the registry implicitly.

**CPS-ART-02 — Verify at deploy** · `MUST` · Baseline
- *What it is.* Signatures and provenance are verified at deploy/admission, not only at build time.
- *Why it's needed.* Build-time signing is moot if deployment accepts anything; admission verification closes the gap.

**CPS-ART-03 — Immutable releases** · `MUST` · Baseline
- *What it is.* Released versions are immutable; mutable references are never deployed to production.
- *Why it's needed.* Mutable tags can be repointed to a malicious artifact after approval.

**CPS-ART-04 — Reproducible builds** · `SHOULD`
- *What it is.* Builds are reproducible from pinned inputs.
- *Why it's needed.* An independent rebuild confirms provenance and supports forensics.

### 4.9 Deployment & Release (REL)
*Controls how artifacts reach production safely — approval, progressive rollout, and rollback.*

| ID | Control | Level | Base |
|---|---|---|---|
| CPS-REL-01 | Pipeline-enforced, tier-appropriate release approval | MUST | B |
| CPS-REL-02 | Short-lived, scoped, federated deploy identity | MUST | B |
| CPS-REL-03 | Progressive multi-tenant deploy with health gates | MUST | B† |
| CPS-REL-04 | Automated, tested rollback | MUST | B |
| CPS-REL-05 | Validate tenant scope/segregation before applying | MUST | B† |

**CPS-REL-01 — Approval gates** · `MUST` · Baseline
- *What it is.* Tier-appropriate release-approval gates are enforced by the pipeline, not by convention.
- *Why it's needed.* Convention-based approval is bypassable; pipeline-enforced gates are not.

**CPS-REL-02 — Federated deploy identity** · `MUST` · Baseline
- *What it is.* Deployment uses short-lived, scoped, federated identity.
- *Why it's needed.* Removes standing production credentials from the single highest-privilege step.

**CPS-REL-03 — Progressive deploy** · `MUST` · Baseline (B†)
- *What it is.* Deployment to shared/multi-tenant targets is progressive, with health gates that halt promotion on failure.
- *Why it's needed.* Bounds blast radius so one bad change cannot reach all tenants at once.

**CPS-REL-04 — Rollback** · `MUST` · Baseline
- *What it is.* Automated rollback is available and tested.
- *Why it's needed.* Fast reversal limits incident duration and improves time-to-restore.

**CPS-REL-05 — Pre-apply validation** · `MUST` · Baseline (B†)
- *What it is.* Before applying any change, the controller validates that the target resolves to the intended tenant scope and preserves the segregation model.
- *Why it's needed.* Last line of defence against deploying into the wrong tenant boundary.

### 4.10 Tenant Segregation (TEN)
*Keeps tenants — internal teams and external customers — isolated from one another.*

| ID | Control | Level | Base |
|---|---|---|---|
| CPS-TEN-01 | Declared, enforced segregation model per tenant | MUST | B |
| CPS-TEN-02 | Cross-team/BU access denied by default (internal) | MUST | B |
| CPS-TEN-03 | Per-external-customer segregation | MUST | B† |
| CPS-TEN-04 | No external/regulated data in shared/lower environments without de-identification | MUST | B† |

**CPS-TEN-01 — Segregation model** · `MUST` · Baseline
- *What it is.* Every tenant has a declared segregation model whose invariants the pipeline enforces.
- *Why it's needed.* Makes isolation explicit and verifiable rather than assumed.

**CPS-TEN-02 — Internal default-deny** · `MUST` · Baseline
- *What it is.* Cross-team/BU access is denied by default; one team's pipeline cannot act on another's resources without explicit grant.
- *Why it's needed.* Prevents lateral movement and accidental cross-team blast radius.

**CPS-TEN-03 — External per-customer segregation** · `MUST` · Baseline (B†)
- *What it is.* Pipelines and credentials delivering to external customers are segregated per customer; no shared credential or resource grants effective cross-customer access.
- *Why it's needed.* A shared credential or resource turns one customer breach into all of them.

**CPS-TEN-04 — Lower-environment data** · `MUST` · Baseline (B†)
- *What it is.* External-customer or regulated data is not present in shared or lower (non-production) environments without approved de-identification.
- *Why it's needed.* The most common real-world isolation failure and a direct privacy/compliance breach.

### 4.11 Logging, Audit & Detection (LOG)
*Provides the audit trail and detection needed to investigate and respond.*

| ID | Control | Level | Base |
|---|---|---|---|
| CPS-LOG-01 | Central append-only audit with tier-based retention | MUST | B |
| CPS-LOG-02 | Cross-platform event normalisation | MUST | |
| CPS-LOG-03 | Monitor/alert privileged & anomalous control-plane actions | MUST | B |

**CPS-LOG-01 — Central audit** · `MUST` · Baseline
- *What it is.* Security-relevant audit events from all platforms go to a central, append-only, access-controlled store with tier-based retention.
- *Why it's needed.* Tamper-evident records are essential for investigation and non-repudiation.

**CPS-LOG-02 — Event normalisation** · `MUST`
- *What it is.* Events are normalised across platforms to enable unified detection and investigation.
- *Why it's needed.* Heterogeneous log formats otherwise create detection blind spots between platforms.

**CPS-LOG-03 — Privileged-action monitoring** · `MUST` · Baseline
- *What it is.* Privileged and anomalous control-plane actions are monitored and alertable.
- *Why it's needed.* Control-plane abuse is how a pipeline compromise escalates; detection enables response.

### 4.12 Resilience & Emergency Change (RES)
*Ensures emergency change is controlled and a compromise can be contained.*

| ID | Control | Level | Base |
|---|---|---|---|
| CPS-RES-01 | Logged, time-boxed break-glass with extra authorisation | MUST | B |
| CPS-RES-02 | Organisation-wide/per-platform deployment freeze (kill switch) | MUST | B |
| CPS-RES-03 | Pipeline configuration recoverable as code | SHOULD | |

**CPS-RES-01 — Break-glass** · `MUST` · Baseline
- *What it is.* A documented break-glass procedure for emergency change: additional authorisation, fully logged, time-boxed, with mandatory post-event review.
- *Why it's needed.* Emergencies will bypass normal gates; an *uncontrolled* bypass is itself the risk, so the bypass must be governed.

**CPS-RES-02 — Kill switch** · `MUST` · Baseline
- *What it is.* An organisation-wide or per-platform deployment freeze is available.
- *Why it's needed.* Containment — the ability to stop the bleeding during a suspected pipeline compromise.

**CPS-RES-03 — Config-as-code recovery** · `SHOULD`
- *What it is.* Pipeline configuration is recoverable/reconstructable as code.
- *Why it's needed.* Enables rebuild after destructive compromise or disaster.

---

## 5. AI/ML Pipeline Controls — MLOps (AI)

*This section applies **only** to pipelines that train, build, or deploy AI/ML models. It extends the general domains to the ML supply chain, where models and datasets are first-class, externally-sourced inputs and where model behaviour cannot be verified by code review alone. Where a control extends a general one, the parent ID is noted. A pipeline that produces or serves a model MUST meet the B-flagged controls here in addition to the general production baseline (§4).*

| ID | Control | Level | Base |
|---|---|---|---|
| CPS-AI-01 | Model & dataset provenance/lineage | MUST | B |
| CPS-AI-02 | Govern external models/datasets as dependencies; scan model files | MUST | B |
| CPS-AI-03 | Separate training pipelines from serving infrastructure | MUST | B |
| CPS-AI-04 | Sign and register model artifacts; verify at promotion | MUST | B |
| CPS-AI-05 | Evaluation & safety gates before promotion | MUST | B |
| CPS-AI-06 | Training-data access governance & provenance | MUST | B |
| CPS-AI-07 | Reproducible model builds | SHOULD | |
| CPS-AI-08 | Model card & intended-use governance | MUST | B |
| CPS-AI-09 | Progressive model rollout & model-aware rollback | MUST | B† |
| CPS-AI-10 | Runtime model & data-plane monitoring | MUST | B |

**CPS-AI-01 — Model & dataset lineage** · `MUST` · Baseline
- *What it is.* Provenance and lineage are maintained for every model and training/evaluation dataset: source, version, licence, and transformations applied.
- *Why it's needed.* Models and data are supply-chain inputs; without lineage you cannot trace a poisoning incident, prove licensing, or reproduce a model. (OWASP LLM Top 10 — LLM03 Supply Chain; NIST AI RMF *Map*.)

**CPS-AI-02 — Govern external models/datasets** · `MUST` · Baseline (extends CPS-DEP-01)
- *What it is.* Base/pretrained models and datasets are sourced only from approved origins, pinned by digest; model files are scanned and unsafe-deserialization formats are rejected or converted before use.
- *Why it's needed.* Third-party weights and data carry poisoning and backdoor risk, and some model file formats execute code on load. (OWASP LLM03/LLM04.)

**CPS-AI-03 — Separate training from serving** · `MUST` · Baseline (extends CPS-EXE-01)
- *What it is.* Training pipelines, which touch data, hold no credentials to tenant-facing serving infrastructure.
- *Why it's needed.* Stops a poisoned or compromised training job from reaching production or tenants.

**CPS-AI-04 — Sign & register models** · `MUST` · Baseline (extends CPS-ART-01/02)
- *What it is.* Model artifacts are signed with provenance and stored in an immutable model registry; promotion verifies the signature.
- *Why it's needed.* Ensures the served model is the evaluated, approved one and has not been swapped or tampered with.

**CPS-AI-05 — Evaluation & safety gates** · `MUST` · Baseline (extends CPS-REL-01)
- *What it is.* Candidate models pass defined evaluation gates — quality/accuracy plus safety/robustness, with bias evaluation at higher tiers — recorded before promotion to serving.
- *Why it's needed.* Model behaviour cannot be code-reviewed; evaluation gates are the equivalent quality and security gate. (NIST AI RMF *Measure*.)

**CPS-AI-06 — Training-data governance** · `MUST` · Baseline (ties CPS-TEN-04)
- *What it is.* Access to training data is governed, least-privilege, and logged; data provenance is captured; sensitive/tenant data is handled per classification.
- *Why it's needed.* Data is the top poisoning and privacy-leak vector and the trace needed to investigate a data-poisoning incident. (OWASP LLM04 Data & Model Poisoning; LLM02 Sensitive Information Disclosure.)

**CPS-AI-07 — Reproducible model builds** · `SHOULD`
- *What it is.* Model builds pin code, a data snapshot, hyperparameters, and the random seed.
- *Why it's needed.* Supports provenance verification, incident forensics, and rollback to a known-good model.

**CPS-AI-08 — Model card & intended use** · `MUST` · Baseline
- *What it is.* Each promotable model has a model card (intended use, evaluation results, limitations); serving a model outside its declared intended use is a policy violation.
- *Why it's needed.* Prevents scope-creep and misuse, and supports accountability and audit.

**CPS-AI-09 — Progressive rollout & model-aware rollback** · `MUST` · Baseline (B†, extends CPS-REL-03/04)
- *What it is.* Models are rolled out progressively (shadow/canary) with monitoring; rollback accounts for model-specific state such as cached embeddings and in-flight traffic.
- *Why it's needed.* Model rollback differs from code rollback, and progressive rollout bounds the blast radius of a regressed or unsafe model.

**CPS-AI-10 — Runtime model monitoring** · `MUST` · Baseline (extends CPS-LOG-03)
- *What it is.* Served models are monitored for drift, degradation, and abuse (e.g., prompt-injection for generative models), with a traceable link to lineage.
- *Why it's needed.* Models degrade over time and the inference data plane is an attack surface; detection enables response. (OWASP LLM01 Prompt Injection; MITRE ATLAS.)

---

## 6. Risk Tiers

**Production baseline** = every control flagged **B** in §4 and §5 (plus **B†** where the target is shared/multi-tenant or external-facing). Tiers escalate requirements *above* baseline.

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
| Model eval gates (CPS-AI-05) | Quality only | + safety/robustness | + bias evaluation + sign-off |
| Key management (CPS-KEY-05) | Platform-managed | Platform-managed | Customer-managed option |
| Audit retention (CPS-LOG-01) | Standard | Extended | Compliance-defined |

---

## 7. Compliance Mapping

Control domains mapped to recognised catalogues for traceability: OWASP Top 10 CI/CD Security Risks, OWASP DevSecOps lifecycle phase, NIST SP 800-53 Rev 5 families, and CSA CCM v4 domains. (Delivery metrics are covered in §8.5.)

| Domain (IDs) | OWASP Top 10 CI/CD | DevSecOps phase | NIST 800-53 Rev 5 | CSA CCM v4 |
|---|---|---|---|---|
| VIS (01–03) | SEC-10 | Operate | CM-8, PM-5 | LOG / GRC |
| IAM (01–08) | SEC-2, SEC-5 | Develop / Operate | AC, IA | IAM |
| SRC (01–05) | SEC-1 | Develop | CM, SA-10, SA-15 | CCC |
| EXE (01–04) | SEC-4 | Build | SC, CM-7 | IVS / AIS |
| KEY (01–05) | SEC-6 | Build / Deploy | IA-5, SC-12, SC-28 | CEK |
| CFG (01–03) | SEC-7 | Build | CM-2, CM-6, CM-7, SC-8 | CCC |
| DEP (01–04) | SEC-3, SEC-8 | Build / Test | SR, SA-12 | STA / AIS |
| ART (01–04) | SEC-9 | Release | SI-7, SR-4, SR-11 | CCC / AIS |
| REL (01–05) | SEC-9 | Deploy | CM-3, CM-4, SA-22 | CCC |
| TEN (01–04) | SEC-2, SEC-5, SEC-7 | Deploy / Operate | SC-2, SC-4, SC-7, AC-4 | IVS / DSP |
| LOG (01–03) | SEC-10 | Operate | AU, SI-4 | LOG |
| RES (01–03) | — | Operate | CP, IR | BCR / SEF |
| AI (01–10) | SEC-3, SEC-9 | Build / Test / Deploy / Operate | SR, SI-7, AC, AU | AIS / STA |

**AI-specific frameworks.** The AI domain (§5) additionally maps to the OWASP Top 10 for LLM Applications (2025) — notably LLM01 (Prompt Injection), LLM03 (Supply Chain), LLM04 (Data & Model Poisoning) — the NIST AI Risk Management Framework, ISO/IEC 42001, and MITRE ATLAS.

---

## 8. Governance, Exceptions & Roles

### 8.1 Ownership
Security Architecture owns, issues, and maintains this standard; defines the baseline and tiers; ratifies versions; and approves exceptions. New repositories, pipelines, platforms, and tenants inherit the baseline by default — opting out requires an exception (§8.4); opting in requires no action.

### 8.2 Roles & responsibilities
| Activity | Security Architecture | Platform owner | Product/Eng team | Tenant owner |
|---|---|---|---|---|
| Own & ratify the standard | **A/R** | C | I | I |
| Define baseline & tiers | **A/R** | C | C | C |
| Implement controls on a platform | C | **A/R** | C | I |
| Maintain conformance register | C | **A/R** | I | I |
| Conform repos/pipelines | I | C | **A/R** | I |
| Inventory & drift detection | **A** | R | C | I |
| Declare tenant segregation needs | C | C | C | **A/R** |
| Govern model/dataset provenance & eval gates | C | C | **A/R** | I |
| Grant/track exceptions | **A/R** | C | C | I |
| Pipeline-compromise response | **A/R** | R | C | I |

(R = Responsible, A = Accountable, C = Consulted, I = Informed.)

### 8.3 Cross-platform conformance & assurance
- Every applicable control MUST be implemented on every in-scope platform using that platform's capabilities; intent is uniform, mechanism may differ.
- Each platform owner MUST maintain a conformance-register entry per control (Appendix A): mechanism and status.
- A native gap MUST be recorded with a compensating control and an exception — never left silent.
- Automated posture/drift detection MUST run across all platforms; overall posture is reported under the weakest-link principle.
- Security Architecture MUST run periodic assurance reviews. Non-conformance without an approved exception is a defect with a tier-based remediation timeline.

### 8.4 Exceptions
- Any unmet MUST requires a formal, documented, risk-assessed, time-boxed exception with a named owner, compensating control, and remediation date, approved by Security Architecture and recorded centrally.
- Tier 3 MUST controls may be excepted only with senior risk-owner sign-off and a hard deadline.
- Expired exceptions are escalated. SHOULD deviations need no exception but SHOULD be documented.

### 8.5 Measurement
Reported with delivery metrics so security and speed stay balanced.
- **Delivery guardrails (DORA, must not be silently degraded):** deployment frequency; lead time for changes; change-failure rate; time to restore service.
- **Security posture metrics:** inventory coverage; baseline conformance rate; open drift detections; federated-identity coverage; standing-credential count (→0); verified-deploy coverage; third-party-component governance coverage; secret-leak MTTR; segregation findings (caught vs escaped); model eval-gate coverage; exception aging; break-glass usage and review-closure rate. Targets MUST carry dates per tier.

---

## 9. References

- OWASP Top 10 CI/CD Security Risks (CICD-SEC-1 … 10) — https://owasp.org/www-project-top-10-ci-cd-security-risks/
- OWASP DevSecOps Guideline — https://owasp.org/www-project-devsecops-guideline/
- OWASP CI/CD Security Cheat Sheet — https://cheatsheetseries.owasp.org/cheatsheets/CI_CD_Security_Cheat_Sheet.html
- OWASP Cheat Sheets cross-referenced for modular topics (this standard adopts OWASP's modular pattern rather than absorbing these in full):
  - Zero Trust Architecture (§3) — https://cheatsheetseries.owasp.org/cheatsheets/Zero_Trust_Architecture_Cheat_Sheet.html
  - Multi-Tenant Security (TEN) — https://cheatsheetseries.owasp.org/cheatsheets/Multi_Tenant_Security_Cheat_Sheet.html
  - Secure AI Model Ops (§5) — https://cheatsheetseries.owasp.org/cheatsheets/Secure_AI_Model_Ops_Cheat_Sheet.html
  - Software Supply Chain Security (DEP / ART) — https://cheatsheetseries.owasp.org/cheatsheets/Software_Supply_Chain_Security_Cheat_Sheet.html
  - Secrets Management and Key Management (KEY) — https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html , https://cheatsheetseries.owasp.org/cheatsheets/Key_Management_Cheat_Sheet.html
- NIST SP 800-53 Rev 5 — Security and Privacy Controls (control families referenced in §7)
- CSA Cloud Controls Matrix (CCM) v4 — cloud security control domains (referenced in §7)
- OWASP Top 10 for LLM Applications (2025) — OWASP GenAI Security Project (referenced in §5/§7)
- NIST AI Risk Management Framework (AI RMF 100-1) and Generative AI Profile
- ISO/IEC 42001 — AI management system
- MITRE ATLAS — adversarial machine-learning threat knowledge base
- DORA (DevOps Research and Assessment) delivery metrics
- Recognised supply-chain integrity-level framework (for ART / AI / Tier 3 provenance bars)

---

## 10. Document Control

| Field | Value |
|---|---|
| Title | CI/CD Security Standard |
| Control-ID namespace | `CPS-` (CI/CD Pipeline Security) |
| Issued by | Security Architecture |
| Status | Draft for review — not yet ratified |
| Version | 0.9 |
| Supersedes | v0.1–0.8 |
| Review cadence | Quarterly, or on material platform/threat change |

**Lifecycle.** Versioning is semantic; material control changes increment the version. Only versions ratified by Security Architecture are binding. On ratification, an adoption timeline by tier is published — new systems conform immediately, existing systems per the timeline.

**Change history.**
| Version | Summary |
|---|---|
| 0.9 | Reworked §3 into Threat Model & Principles (Zero Trust folded in as the lens, CI/CD risk framing added) per OWASP CI/CD Security Cheat Sheet; closed coverage gaps (CPS-IAM-08 MFA, CPS-CFG-03 secure transport, enriched CPS-EXE-03 and CPS-SRC-01); adopted OWASP's modular cross-referencing in §9; added Appendix B coverage map |
| 0.8 | Added per-control "what it is / why it's needed" detail with a summary table per domain; added §5 AI/ML Pipeline Controls (MLOps) with `CPS-AI-NN`; added AI frameworks to the crosswalk and references; renumbered later sections |
| 0.7 | Adopted `CPS-<DOMAIN>-NN` IDs; folded baseline into §4 as a flag; replaced §6 DORA column with NIST 800-53 + CCM v4 crosswalk |
| 0.6 | Restructured into the concise 9-section form; controls converted to ID'd domain tables |
| 0.1–0.5 | Earlier drafts (initial scope, vendor-neutral rewrite, plain-language pass) |

**Note on "DORA".** Used here as the DevOps Research and Assessment delivery metrics. If the EU Digital Operational Resilience Act is also meant, §6, §8.3, and §8.5 are the hooks for a separate regulatory-mapping annex.

---

## Appendix A — Per-platform conformance register (template)

Maintained by platform owners; one column per in-scope platform. Status: ✅ native / 🟡 compensating + exception / ❌ gap. Extend rows to cover all applicable controls (including §5 AI controls for ML pipelines).

| Control ID | Platform 1 (mechanism) | Status | Platform 2 (mechanism) | Status | Platform n … |
|---|---|---|---|---|---|
| CPS-IAM-01 | | | | | |
| CPS-IAM-02 | | | | | |
| CPS-IAM-08 | | | | | |
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
| CPS-AI-02 | | | | | |
| CPS-AI-04 | | | | | |
| CPS-AI-05 | | | | | |

---

## Appendix B — Coverage map: OWASP CI/CD Security Cheat Sheet → this standard

Shows that the control areas emphasised by the OWASP CI/CD Security Cheat Sheet are fully covered, and where this standard goes beyond it.

| OWASP Cheat Sheet area | This standard's domains |
|---|---|
| Secure Configuration (SCM config + pipeline/execution environment) | SRC, EXE, CFG |
| IAM (secrets management, least privilege, identity lifecycle) | IAM, KEY |
| Managing Third-Party Code (dependencies, plug-ins/integrations) | DEP |
| Integrity Assurance | ART |
| Visibility and Monitoring | LOG |
| *Beyond the cheat sheet (added by this standard)* | VIS, REL, TEN, RES, AI (MLOps), and the governance/tiering/measurement apparatus |
