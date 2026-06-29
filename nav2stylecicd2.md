# CI/CD Security Standard for an Organization

**Issued by:** Security Architecture  **Status:** Draft for review (not ratified)  **Version:** 1.1

**Conventions.** **shall / shall not** = mandatory; **should / should not** = recommended. An unmet *shall* requires a documented, time-bound exception (§3.15, Annex A). Full metadata and change history are in Annex C.

---

## 1. Purpose

The purpose of this standard is to define mandatory security controls for software delivery pipelines from source-code commit through build, test, artifact creation, release, deployment, and production monitoring.

A CI/CD security standard is required because CI/CD systems are part of the organization's software supply chain and can directly influence production code, infrastructure, secrets, artifacts, and deployments. NIST SP 800-204D focuses on integrating software supply-chain security into DevSecOps CI/CD pipelines and maps those strategies to SSDF practices. A weak pipeline can bypass many traditional controls because it can produce signed binaries, container images, infrastructure changes, and production deployments directly; the standard is therefore risk-based, auditable, and Zero Trust-aligned, applied consistently across every platform in use and every tenant served.

This standard shall apply to:

- Source-code repositories.
- CI/CD platforms and orchestrators.
- Build agents and runners.
- Pipeline definitions and templates.
- Artifact repositories and package feeds.
- Container registries.
- Infrastructure-as-Code pipelines.
- Deployment automation.
- Secrets, certificates, signing keys, and workload identities.
- Third-party integrations and open-source dependencies.
- Pipelines that train, build, or deploy AI/ML models (§3.18).

Out of scope (governed elsewhere): developer endpoint security; application/model runtime security beyond what the pipeline deploys; production infrastructure not provisioned via CI/CD; corporate identity governance (a dependency of §3.2); data classification (a dependency of §3.16). The standard applies regardless of hosting model; third-party-operated platforms shall be brought into conformance contractually.

---

## 2. Security Principles

Every CI/CD implementation shall follow these principles.

| Principle | Standard requirement |
|---|---|
| Zero Trust | Every user, workload identity, agent, repository, artifact, and deployment path shall be explicitly verified before access or execution; no implicit trust from network location or a passing build. |
| Least Privilege | Pipeline identities, service connections, repositories, and deployment roles shall hold only the permissions required for the specific job, time-bound where possible. |
| Secure by Default | Security checks shall be enabled by default and shall not rely on manual developer action. |
| Defense in Depth | Controls shall exist at source, build, dependency, artifact, deployment, and runtime layers. |
| Tamper Resistance | Build outputs, images, packages, and deployment artifacts shall be traceable, signed, and protected from unauthorized modification. |
| Continuous Verification | Security posture shall be validated continuously through automated scanning, policy gates, monitoring, and audit evidence. |
| Separation of Duties | Developers, approvers, release owners, and exception approvers shall not have unrestricted overlapping control over production releases. |
| Consistency Across Platforms | One control intent shall apply to every platform; overall posture is reported at the level of the least-conformant component (the weakest-link principle, §3.17). |

---

## 3. Mandatory CI/CD Security Control Domains

Each domain states a **Standard**, lists **Mandatory controls**, notes the framework basis, and defines **Minimum evidence**.

### 3.1 Source Code and Repository Security
**Standard.** Repositories used for production or production-connected workloads shall enforce protected branches, peer review, access control, and secure change management.

**Mandatory controls.**
- Direct commits to protected branches such as main, release, and production shall be blocked.
- Pull requests shall require review by a non-author before merge.
- Branch protection shall require successful build and security checks before merge.
- Administrative bypass of required checks shall be restricted, approved, and logged.
- Pipeline definition and workflow files shall be treated as security-sensitive code and reviewed before change.
- Repositories shall have an assigned owner, with access reviewed periodically.
- Secret scanning shall run pre-merge; detections shall block the merge and trigger rotation.
- Auto-merge that bypasses review shall be disabled; forking of private/internal repositories and changing repository visibility to public shall be restricted.
- Commits and tags should be signed, and unsigned changes should be kept out of production.

OWASP CI/CD Top 10 identifies insufficient flow control mechanisms as a key risk; NIST SSDF and the SLSA Source track address source integrity and review.

**Minimum evidence.** Branch-protection configuration; pull-request approval logs; repository access-review records; secret-scan results; pipeline-file change history.

### 3.2 Identity and Access Management for Pipelines
**Standard.** CI/CD access shall be identity-based, least-privilege, strongly authenticated, and separated across development, test, staging, and production.

**Mandatory controls.**
- Human access to CI/CD platforms shall require MFA.
- Humans shall authenticate via the central identity provider (SSO); standalone or local accounts shall not be used for standing access.
- Joiner/mover/leaver lifecycle shall be federated so that one deprovisioning action removes access across all platforms.
- Access shall be least-privilege, granted to roles or groups, and recertified periodically.
- Privileged CI/CD administration shall use just-in-time or time-bound access where available.
- Production deployment identities shall be separate from build identities; service accounts shall not be shared across unrelated applications or environments.
- Long-lived credentials shall be avoided; workload identity federation or managed identities shall be used where supported.
- Service principals, PATs, deploy keys, and tokens shall have expiry and rotation.
- Non-human identities shall be inventoried, owned, and scoped.
- Segregation of duties shall separate policy/exception authoring from production deployment, and release approval shall be restricted to authorized roles.
- Each pipeline's privilege shall be scoped to its target resources or tenant, time-bound and revocable.
- External collaborators shall be provisioned with explicit scope and expiry, with no default standing access.

OWASP CI/CD Top 10 identifies inadequate identity and access management and insufficient pipeline-based access controls as key risks.

**Minimum evidence.** RBAC assignment export; service-connection inventory and owners; token expiry/rotation report; privileged-access logs; approval-role configuration.

### 3.3 Secrets, Keys, Certificates, and Credential Hygiene
**Standard.** Secrets shall never be stored in source code, pipeline definitions, build logs, container images, artifacts, scripts, or package metadata.

**Mandatory controls.**
- Secrets shall be stored only in approved secret stores or vaults.
- Pipeline logs shall mask secrets; secrets shall not be printed, echoed, cached, or included in artifacts.
- Secret scanning shall run on repositories and pipelines.
- Production and non-production secrets shall be separated.
- Secrets shall have defined owners, expiry, and rotation.
- No standing long-lived production or cloud credentials shall be held; short-lived federated workload identity shall be used.
- Secrets and keys shall be scoped so that no single credential grants effective cross-tenant access.
- Code-signing keys and high-value TLS private keys shall use HSM-backed or equivalent protected storage where required; private keys shall not be exported, logged, or written to artifacts, image layers, or pipeline output.
- Certificate issuance, renewal, revocation, and key-access events shall be logged.
- Customer-managed keys should be available where external-customer or regulatory requirements apply.

OWASP CI/CD Top 10 identifies insufficient credential hygiene as a key risk; NIST SSDF and ISO/IEC 27001:2022 cryptography controls address secret and key protection.

**Minimum evidence.** Secret-store/vault inventory; rotation records; secret-scan results; log-masking validation; certificate issuance and renewal logs.

### 3.4 Build Environment Security
**Standard.** Build environments shall be hardened, isolated, reproducible, and protected from unauthorized network, dependency, or toolchain manipulation.

**Mandatory controls.**
- Production builds shall run on approved build agents or runners only.
- Self-hosted agents shall follow an approved OS hardening baseline and be patched, monitored, and isolated.
- Build environments shall be isolated per job; shared agents shall not retain secrets, artifacts, or workspace residue after completion.
- Build agents shall not run with unnecessary administrative or root privilege; privileged containers shall not be used for builds unless explicitly approved.
- Untrusted pipelines (running un-reviewed external input) shall be separated from trusted pipelines so untrusted input cannot reach production or cross-tenant credentials.
- Build environments shall restrict outbound network access where feasible.
- Build tools, compilers, SDKs, and base images shall come from approved sources.
- Build pipelines shall be version-controlled and generate auditable logs.

OWASP CI/CD Top 10 identifies poisoned pipeline execution and insecure system configuration as key risks; SLSA Build levels and NIST SP 800-204D address build-environment integrity.

**Minimum evidence.** Approved agent-pool configuration; agent hardening baseline; network-isolation policy; toolchain source records; build logs and attestations.

### 3.5 Dependency and Package Supply Chain Security
**Standard.** All third-party and open-source dependencies shall be governed, scanned, approved, and traceable.

**Mandatory controls.**
- Dependencies shall be pulled only from approved package feeds or registries.
- Public package registries shall not be used directly in production builds unless explicitly approved.
- Dependency lock files shall be used where supported.
- Dependency scanning shall run on every pull request and build.
- High and critical dependency vulnerabilities shall block release unless an approved exception exists.
- Dependency-confusion protection shall be enforced.
- Package-feed permissions should be least-privilege and component ownership tracked.

OWASP CI/CD Top 10 identifies dependency chain abuse as a major risk; NIST SP 800-204D and CIS Software Supply Chain guidance address approved-source consumption.

**Minimum evidence.** Approved-feed list; dependency-scan results; SBOM (§3.9); vulnerability-remediation records; exception approvals.

### 3.6 Static, Dynamic, and Software Composition Security Testing
**Standard.** Security testing shall be automated and embedded into CI/CD pipelines with severity-based gates.

**Mandatory controls.**
- Static Application Security Testing (SAST) shall run for supported languages.
- Software Composition Analysis (SCA) shall run for open-source components.
- Secret scanning shall run on source and pipeline context.
- Infrastructure-as-Code scanning shall run for Terraform, Bicep, ARM, CloudFormation, Kubernetes YAML, Helm, and equivalent.
- Container image scanning shall run before registry push and before deployment.
- Findings shall be classified by severity, and critical unresolved findings shall block release unless an approved exception exists.
- Dynamic Application Security Testing (DAST) should be performed for applicable web and API workloads.

NIST SSDF requires producing well-secured software; automated scanning gates mitigate OWASP CI/CD risks across dependencies, credentials, and configuration.

**Minimum evidence.** SAST, SCA, secret, IaC, and container scan reports; release-gate logs; finding-to-closure tracking.

### 3.7 Pipeline Flow Control and Approval Gates
**Standard.** Software shall move from development to production through controlled, enforced stages.

**Mandatory controls.**
- Pipelines shall define separate stages for build, test, security validation, staging, and production.
- Production deployment shall require approval gates.
- Developers shall not be able to unilaterally bypass security checks for production; self-approval and check-bypass shall be disabled except via break-glass.
- Emergency releases shall follow a documented break-glass path with additional authorization, full logging, time-boxing, and post-release review.
- Release pipelines shall preserve evidence of what was deployed, who approved it, and when.

OWASP CI/CD Top 10 identifies insufficient flow control mechanisms as a key risk.

**Minimum evidence.** Stage definitions; approval logs; security-gate results; emergency-change records; deployment audit logs.

### 3.8 Artifact Integrity, Signing, and Provenance
**Standard.** All production artifacts shall be traceable to source, build process, dependencies, and the approving pipeline.

**Mandatory controls.**
- Build artifacts and released versions shall be immutable after creation.
- Deployment shall use artifacts produced by approved CI pipelines only.
- Artifacts shall be signed where supported.
- Container images shall be referenced by digest, not mutable tags, for production deployment.
- Build provenance or attestation shall capture source commit, builder identity, build parameters, and artifact digest.
- Artifact repositories shall enforce retention, immutability, and access control.
- Deployment systems shall verify artifact signatures and integrity before deployment.
- Builds should be reproducible to support provenance verification.
- Provenance assurance should increase with maturity (SLSA Build Track L1 through L3; see §5).

OWASP CI/CD Top 10 identifies improper artifact integrity validation as a key risk; SLSA and NIST SP 800-204D define provenance and attestation.

**Minimum evidence.** Artifact digest; signing record; provenance attestation; artifact-repository ACL; deployment verification logs.

### 3.9 SBOM and Component Inventory
**Standard.** A Software Bill of Materials shall be generated and retained for production software.

**Mandatory controls.**
- An SBOM shall be generated for production releases.
- The SBOM shall include direct and transitive dependencies where supported.
- The SBOM shall be associated with release version, artifact digest, and build identifier.
- The SBOM shall be stored in an approved repository and available for vulnerability impact analysis and incident response.
- SBOM generation should be automated in CI/CD where feasible.

NIST SP 800-204D and CIS Software Supply Chain guidance treat the SBOM as core supply-chain visibility.

**Minimum evidence.** SBOM file; release-to-SBOM mapping; artifact-digest mapping; SBOM storage location; vulnerability impact-analysis record.

### 3.10 Infrastructure-as-Code and Environment Security
**Standard.** Infrastructure deployed through CI/CD shall be defined, reviewed, scanned, and governed as code.

**Mandatory controls.**
- Infrastructure changes shall be made through version-controlled IaC.
- IaC shall be scanned before deployment.
- Cloud resource configurations shall be validated against organizational baselines.
- Production infrastructure changes shall require review and approval.
- Privileged cloud permissions used by pipelines shall be minimized.
- Network, identity, encryption, logging, and exposure controls shall be policy-validated before deployment.
- Configuration drift detection should be enabled where feasible.

OWASP CI/CD Top 10 identifies insecure system configuration as a key risk; NIST SP 800-204D addresses secure build and deployment of infrastructure.

**Minimum evidence.** IaC scan report; policy-compliance report; deployment approval; drift-detection record; cloud posture status.

### 3.11 Container and Kubernetes Pipeline Security
**Standard.** Containerized workloads shall be built and deployed using secure image, registry, and runtime practices.

**Mandatory controls.**
- Container images shall use approved base images.
- Images shall be scanned for vulnerabilities before push and before deployment.
- Images shall be signed where supported.
- Production deployments shall use trusted registries only.
- Images shall run as non-root where possible; privileged containers shall be prohibited unless explicitly approved.
- Admission controls shall restrict unsigned, unscanned, or untrusted images.
- Kubernetes manifests and Helm charts shall be scanned as IaC.

CIS Kubernetes and container benchmarks and OWASP artifact-integrity guidance address image and runtime security.

**Minimum evidence.** Image scan results; image-signing logs; base-image approval; registry policy; admission-control configuration.

### 3.12 Deployment Security and Release Governance
**Standard.** Production deployment shall be controlled, observable, reversible, and auditable.

**Mandatory controls.**
- Production deployment shall be automated through approved release pipelines.
- Manual production changes shall be prohibited unless covered by the emergency process.
- Deployment shall use short-lived, scoped, federated identity.
- Deployment shall support rollback or forward-fix, and rollback shall be tested.
- Deployment health checks shall be defined.
- Production approval and deployment logs shall record artifact version, approver, environment, and result.
- Segregation shall exist between development, staging, and production.
- For shared or multi-tenant targets, deployment shall be progressive with health gates, and target tenant scope and segregation shall be validated before applying any change.
- An organization-wide or per-platform deployment freeze (kill switch) shall be available to contain a suspected compromise.

OWASP CI/CD Top 10 identifies insufficient flow control as a key risk; NIST SP 800-204D addresses secure release governance.

**Minimum evidence.** Deployment-pipeline logs; approval-gate records; rollback test records; environment-separation evidence; production change record.

### 3.13 Logging, Monitoring, and Detection
**Standard.** CI/CD platforms shall generate security-relevant logs and forward them to centralized monitoring or SIEM.

**Mandatory controls.**
- Repository, pipeline-run, approval/rejection/bypass/exception, secret-access, artifact (create, sign, publish, delete), and deployment events shall be logged.
- Logs shall be forwarded to centralized monitoring or SIEM and normalized across platforms.
- Logs shall be protected from tampering (append-only, access-controlled, with defined retention).
- Detection rules shall cover suspicious CI/CD activity, and privileged or anomalous control-plane actions shall be alerted.

OWASP CI/CD Top 10 identifies insufficient logging and visibility as a key risk.

**Minimum evidence.** SIEM ingestion proof; audit-log export; detection-rule list; incident-investigation sample; log-retention configuration.

### 3.14 Third-Party Integrations and Marketplace Actions
**Standard.** Third-party pipeline tasks, actions, extensions, and integrations shall be governed before use.

**Mandatory controls.**
- Third-party CI/CD actions, tasks, and extensions shall be approved before use.
- Third-party actions shall be version-pinned to an immutable reference.
- Untrusted actions shall not receive secrets and shall not run in trusted pipelines.
- Integrations shall be reviewed for permissions and data access, and marketplace extensions reviewed before installation.
- Vendor-compromise and package-compromise response procedures shall be defined.

OWASP CI/CD Top 10 identifies ungoverned usage of third-party services as a key risk.

**Minimum evidence.** Approved-integration inventory; permission review; version-pinning evidence; vendor risk approval; exception records.

### 3.15 Exception Management
**Standard.** Security exceptions shall be formal, time-bound, risk-accepted, and reviewed.

**Mandatory controls.**
- Exceptions shall record the control, affected system, risk, compensating control, owner, approver, and expiry.
- Exceptions shall not be permanent by default.
- Critical release-blocking exceptions shall require security and business approval.
- Expired exceptions shall automatically trigger review.
- Exception trends shall be reported to security governance.

Risk-based governance under NIST and ISO/IEC 27001:2022 requires documented, time-bound exceptions with named ownership.

**Minimum evidence.** Exception register; exception approvals; expiry reviews; compensating-control evidence; governance report.

### 3.16 Tenant Segregation — Internal and External
**Standard.** Tenants — internal teams or business units and external customers or partners — shall be isolated from one another across the pipeline.

**Mandatory controls.**
- Every tenant shall have a declared segregation model whose invariants the pipeline enforces.
- Cross-team or cross-BU access shall be denied by default.
- Pipelines and credentials delivering to external customers shall be segregated per customer; no shared credential or resource shall grant effective cross-customer access.
- External-customer or regulated data shall not be present in shared or lower (non-production) environments without approved de-identification.

Tenant isolation aligns to NIST SP 800-53 boundary and isolation controls and CSA CCM, and addresses OWASP access-management and configuration risks in a multi-tenant context.

**Minimum evidence.** Segregation-model declarations; per-tenant credential and key scoping; deploy-time segregation validation; lower-environment data-handling attestations.

### 3.17 Estate Inventory and Cross-Platform Conformance
**Standard.** Across a heterogeneous, multi-platform toolchain, control intent shall be applied consistently on every platform, and posture reported at the weakest link.

**Mandatory controls.**
- A continuous inventory of all platforms, organizations/projects, repositories, and pipelines shall be maintained; unknown repositories are non-conformant.
- New organizations, projects, or repositories shall be detected within a defined window.
- Every applicable control shall be implemented on every in-scope platform using that platform's capabilities; a native gap shall be recorded with a compensating control and an exception.
- Each platform owner shall maintain a conformance-register entry per control (Annex D).
- Automated posture and drift detection shall run across all platforms; overall posture is reported under the weakest-link principle.

Consistent multi-platform control and visibility address OWASP insecure-configuration and insufficient-logging risks across the estate.

**Minimum evidence.** Estate inventory; conformance register; drift-detection findings; periodic assurance-review reports.

### 3.18 AI/ML Pipeline Security — MLOps
**Standard.** Pipelines that train, build, or deploy AI/ML models shall extend the supply-chain controls to models and datasets, whose behaviour code review cannot fully verify. This domain applies only to model-producing or model-serving pipelines, in addition to the general controls.

**Mandatory controls.**
- Model and dataset provenance and lineage (source, version, licence, transformations) shall be maintained.
- Base models and datasets shall be sourced from approved origins, pinned by digest; model files shall be scanned and unsafe-deserialization formats rejected or converted.
- Training pipelines shall hold no credentials to tenant-facing serving infrastructure.
- Model artifacts shall be signed with provenance, stored in an immutable registry, and verified at promotion.
- Candidate models shall pass evaluation gates — quality plus safety and robustness, with bias evaluation for higher-assurance or external-facing models — recorded before promotion.
- Training-data access shall be governed, least-privilege, and logged; data provenance shall be captured; sensitive or tenant data shall be handled per classification.
- Each promotable model shall have a model card (intended use, evaluation results, limitations); serving a model outside its declared intended use is a violation.
- Models shall be rolled out progressively (shadow or canary) with monitoring, and rollback shall account for model-specific state.
- Served models shall be monitored for drift, degradation, and abuse (such as prompt-injection), linked to lineage.
- Model builds should be reproducible (pinned code, data snapshot, hyperparameters, and seed).

NIST SP 800-218A (the SSDF profile for generative AI and dual-use foundation models), the OWASP Top 10 for LLM Applications (2025), the NIST AI Risk Management Framework, ISO/IEC 42001, and MITRE ATLAS address the ML supply chain.

**Minimum evidence.** Model and dataset lineage records; model-registry and signing logs; evaluation and safety-gate results and model cards; training-data access logs; rollout, rollback, and monitoring records.

---

## 4. Minimum Enterprise Baseline — Non-Negotiable Controls

For enterprise adoption, the following shall be mandatory for any production or production-connected pipeline:

- A CI/CD pipeline inventory shall be maintained and each pipeline shall have an assigned owner.
- Protected branches shall be enabled and direct commits to production branches blocked.
- Pull-request review shall be required before merge, and pipeline-definition changes shall be reviewed with the same rigor as code.
- MFA and least-privilege RBAC shall be enforced for CI/CD platforms, and production deployment access separated from build access.
- No secrets shall be hardcoded in code, pipeline definitions, scripts, logs, or artifacts; secrets shall be stored in approved vaults.
- No standing production or cloud credentials shall be held; federated workload identity shall be used.
- SAST, SCA, secret scanning, IaC scanning, and container scanning shall be automated in pipelines, with critical findings blocking release unless formally excepted.
- Production builds shall consume dependencies only from approved sources.
- An SBOM shall be generated for production releases.
- Production artifacts shall be immutable, signed or integrity-protected where supported, and build provenance captured.
- Production deployment shall require approval gates and an audit trail, with tested rollback; for shared or external-facing targets, deployment shall be progressive and tenant-scoped.
- CI/CD security logs shall be forwarded to SIEM or centralized logging and protected from tampering.
- Third-party actions shall be approved and version-pinned, and a deployment freeze shall be available.
- Exceptions shall be time-bound, risk-accepted, and reviewed.
- AI/ML pipelines shall additionally meet the controls in §3.18.

---

## 5. CI/CD Standard Maturity Model

| Level | Description | Expected controls |
|---|---|---|
| Level 0 — Uncontrolled | Manual releases, weak access, no consistent scanning | Not acceptable for production |
| Level 1 — Basic Hygiene | Branch protection, code review, basic SAST/SCA | Minimum for low-risk internal apps |
| Level 2 — Controlled | Secrets vaulting, IaC and container scanning, approval gates | Minimum for production workloads |
| Level 3 — Supply-Chain Assured | SBOM, signed artifacts, provenance, trusted dependencies, SLSA Build L2+ | Required for regulated or high-value services |
| Level 4 — Advanced Zero Trust CI/CD | Hermetic/isolated builds, network isolation, policy-as-code, continuous detection, SLSA Build L3 | Target state for critical or external-facing platforms |

Maturity describes organizational capability progression; required control intensity increases with level, and external-facing or regulated services are expected to operate at Level 3 or above.

---

## 6. Policy Wording You Can Use Directly

> All organization-owned CI/CD pipelines that build, test, package, sign, publish, or deploy software to production or production-connected environments — including pipelines that train, build, or deploy AI/ML models — shall implement mandatory security controls across source control, identity and access, secrets and key management, build-environment hardening, dependency governance, security testing, artifact integrity and provenance, SBOM, infrastructure-as-code, container and Kubernetes security, deployment approvals, logging and monitoring, third-party governance, tenant segregation, and exception management. Pipelines shall enforce Zero Trust, least privilege, secure-by-default configuration, tamper-resistant artifact handling, and auditable release governance, applied consistently across every platform in use. Any deviation shall require documented risk acceptance, a compensating control, named ownership, and an expiry date.

---

## 7. Control Mapping to Standards

| CI/CD control area | NIST SSDF / SP 800-204D | OWASP CI/CD Top 10 | SLSA | CIS / ISO |
|---|---|---|---|---|
| Source control (3.1) | Secure development practices | Insufficient flow control | Source track | CIS SSC; ISO 27001 A.8.4 |
| Pipeline IAM (3.2) | Secure environment | Inadequate IAM; insufficient PBAC | Build platform trust | ISO 27001 A.5.15/A.5.18 |
| Secrets & keys (3.3) | Protect code and credentials | Credential hygiene | Build integrity | ISO 27001 A.8.24/A.5.33 |
| Build environment (3.4) | CI/CD supply-chain strategies | Poisoned pipeline execution | Build L2–L3 | CIS SSC; ISO 27001 A.8.31 |
| Dependencies (3.5) | Supply-chain security (800-204D) | Dependency chain abuse | Dependency assurance | CIS SSC; ISO 27034 |
| Security testing (3.6) | Produce well-secured software | Dependency/credential/config | — | ISO 27034 |
| Flow control (3.7) | Secure release | Insufficient flow control | — | ISO 27001 A.8.32 |
| Artifact integrity (3.8) | Verify integrity (800-204D) | Artifact integrity validation | Provenance / attestation | ISO 27001 A.8.9 |
| SBOM (3.9) | Supply-chain visibility | Dependency risk | Complements provenance | CIS SSC; ISO 27001 A.5.9 |
| IaC & environment (3.10) | Secure build and deployment | Insecure configuration | — | ISO 27001 A.8.9 |
| Container & Kubernetes (3.11) | Secure runtime/deployment | Artifact integrity | — | CIS Kubernetes Benchmark |
| Deployment governance (3.12) | Secure release governance | Insufficient flow control | — | ISO 27001 A.8.32 |
| Logging & detection (3.13) | Secure operations | Insufficient logging | Auditability | ISO 27001 A.8.15/A.8.16 |
| Third-party integrations (3.14) | Supply-chain risk management | Ungoverned third-party services | — | ISO 27001 A.5.19/A.5.21 |
| Exception management (3.15) | Risk-based governance | All risk classes | — | ISO 27001 Clause 6 / A.5 |
| Tenant segregation (3.16) | Secure environment | IAM / PBAC / configuration | — | ISO 27001 A.8.22 |
| Cross-platform conformance (3.17) | Secure operations | Insecure config; insufficient logging | — | ISO 27001 A.5.9 |
| AI/ML pipeline (3.18) | SP 800-218A | LLM01 / LLM03 / LLM04 | Provenance | ISO/IEC 42001; NIST AI RMF |

A supplementary crosswalk to NIST SP 800-53 Rev 5 control families and CSA CCM v4 is available on request for enterprise GRC.

---

## 8. Security Architect Position

CI/CD security is not only scanning code; it is a full software supply-chain control plane. The standard must protect who can change code, who can change pipelines, what dependencies are allowed, where secrets are stored, how builds are isolated, how artifacts are verified, who can approve production, what evidence proves compliance, and how quickly risks are detected and remediated. Because a weak pipeline can directly produce signed binaries, container images, infrastructure changes, and production deployments, the organization's CI/CD standard shall be treated as a **Tier-0 engineering security standard**, not a DevOps best-practice document.

**Final one-line definition.** *An organizational CI/CD Security Standard is a mandatory, auditable, Zero Trust-aligned security baseline — applied consistently across every platform and tenant — that protects the software delivery pipeline from unauthorized code changes, credential leakage, dependency compromise, build tampering, artifact manipulation, and unsafe production deployment.*

---

## Annex A — Governance, Roles, Exceptions, and Measurement

**Ownership.** Security Architecture owns, issues, and maintains this standard; defines the baseline; ratifies versions; and approves exceptions. New repositories, pipelines, platforms, and tenants inherit the baseline by default; opting out requires an exception (§3.15).

**Roles and responsibilities.**
| Activity | Security Architecture | Platform owner | Product/Eng team | Tenant owner |
|---|---|---|---|---|
| Own and ratify the standard | A/R | C | I | I |
| Define the baseline | A/R | C | C | C |
| Implement controls on a platform | C | A/R | C | I |
| Maintain the conformance register | C | A/R | I | I |
| Conform repositories/pipelines | I | C | A/R | I |
| Inventory and drift detection | A | R | C | I |
| Declare tenant segregation needs | C | C | C | A/R |
| Govern model/dataset provenance and eval gates | C | C | A/R | I |
| Grant and track exceptions | A/R | C | C | I |
| Pipeline-compromise response | A/R | R | C | I |

(R = Responsible, A = Accountable, C = Consulted, I = Informed.)

**Conformance and assurance.** Every applicable control shall be implemented on every in-scope platform (§3.17), verified from automated policy evaluation and audit logs for technical controls and from recorded attestation or review for process controls. Security Architecture shall run periodic assurance reviews and report posture under the weakest-link principle; non-conformance without an approved exception is a defect with a remediation timeline.

**Measurement.** Reported with delivery metrics so security and speed stay balanced. Delivery guardrails (DORA), not to be silently degraded: deployment frequency; lead time for changes; change-failure rate; time to restore service. Security posture metrics: inventory coverage; baseline conformance rate; open drift detections; federated-identity coverage; standing-credential count (target zero); verified-deploy coverage; third-party-component governance coverage; secret-leak mean-time-to-rotate; segregation findings (caught versus escaped); model eval-gate coverage; exception aging; break-glass usage and review-closure rate. Targets shall carry dates.

---

## Annex B — References

- OWASP Top 10 CI/CD Security Risks — https://owasp.org/www-project-top-10-ci-cd-security-risks/
- OWASP CI/CD Security Cheat Sheet — https://cheatsheetseries.owasp.org/cheatsheets/CI_CD_Security_Cheat_Sheet.html
- OWASP DevSecOps Guideline — https://owasp.org/www-project-devsecops-guideline/
- NIST SP 800-218 (SSDF v1.1) and NIST SP 800-218A (SSDF profile for generative AI and dual-use foundation models, 2024)
- NIST SP 800-204D (integrating software supply-chain security into CI/CD)
- SLSA — Supply-chain Levels for Software Artifacts, Build Track L0–L3 — https://slsa.dev/
- CIS Software Supply Chain Security Guide and CIS Benchmarks (including GitHub, GitLab, Kubernetes)
- ISO/IEC 27001:2022 (Annex A controls) and ISO/IEC 27034 (application security)
- OWASP Top 10 for LLM Applications (2025); NIST AI Risk Management Framework; ISO/IEC 42001; MITRE ATLAS (for §3.18)
- DORA (DevOps Research and Assessment) delivery metrics
- OWASP modular cheat sheets cross-referenced: Zero Trust Architecture; Multi-Tenant Security; Secure AI Model Ops; Software Supply Chain Security; Secrets Management; Key Management — https://cheatsheetseries.owasp.org/

---

## Annex C — Document Control

| Field | Value |
|---|---|
| Title | CI/CD Security Standard for an Organization |
| Issued by | Security Architecture |
| Status | Draft for review — not yet ratified |
| Version | 1.1 |
| Supersedes | v0.1–v1.0 |
| Review cadence | Quarterly, or on material platform or threat change |

**Lifecycle.** Versioning is semantic; material control changes increment the version. Only versions ratified by Security Architecture are binding. On ratification, an adoption timeline is published.

**Change history.**
| Version | Summary |
|---|---|
| 1.1 | Restructured to mirror the NV1 layout: eight-section spine, per-domain Standard / Mandatory controls (plain bullets) / framework basis / Minimum evidence. Removed control-ID, Level, and Base columns; removed the Risk Tiers section (differentiation now via the Maturity Model); reduced the control-mapping table to five columns; updated Purpose and per-domain wording to the NV1 framing. Governance, references, document control, and appendices moved to trailing annexes. Vendor-neutral; "shall/should" wording. |
| 1.0 | NV1-aligned content with per-domain tables, IDs, tiers, and supplementary mapping column |
| 0.1–0.9 | Earlier drafts |

---

## Annex D — Per-platform conformance register (template)

Maintained by platform owners; one column per in-scope platform. Status: ✅ native / 🟡 compensating control + exception / ❌ gap. Extend rows to cover all applicable controls.

| Control (domain) | Platform 1 (mechanism) | Status | Platform 2 (mechanism) | Status | Platform n … |
|---|---|---|---|---|---|
| MFA for human access (3.2) | | | | | |
| Federated joiner/mover/leaver (3.2) | | | | | |
| Protected branches (3.1) | | | | | |
| Secrets in approved vault (3.3) | | | | | |
| Untrusted/trusted pipeline separation (3.4) | | | | | |
| Approved dependency sources (3.5) | | | | | |
| SAST/SCA/secret/IaC/container scanning (3.6) | | | | | |
| Artifact signing and provenance (3.8) | | | | | |
| SBOM generation (3.9) | | | | | |
| Admission control for images (3.11) | | | | | |
| Progressive multi-tenant deploy (3.12) | | | | | |
| Central log forwarding (3.13) | | | | | |
| Third-party action pinning (3.14) | | | | | |
| Tenant segregation enforcement (3.16) | | | | | |
| Model signing and eval gates (3.18) | | | | | |

---

## Annex E — Coverage maps

**OWASP CI/CD Security Cheat Sheet → this standard**

| Cheat-sheet area | Domains |
|---|---|
| Secure Configuration (SCM + pipeline/execution environment) | 3.1, 3.4, 3.10 |
| IAM (secrets, least privilege, identity lifecycle) | 3.2, 3.3 |
| Managing Third-Party Code (dependencies, plug-ins) | 3.5, 3.14 |
| Integrity Assurance | 3.8, 3.9 |
| Visibility and Monitoring | 3.13, 3.17 |
| Beyond the cheat sheet | 3.6, 3.7, 3.11, 3.12, 3.15, 3.16, 3.18 |

**NV1 (3.1–3.15) → this standard.** Source→3.1, IAM→3.2, Secrets→3.3, Build→3.4, Dependency→3.5, Testing→3.6, Flow→3.7, Artifact→3.8, SBOM→3.9, IaC→3.10, Container→3.11, Deployment→3.12, Logging→3.13, Third-Party→3.14, Exception→3.15. Added beyond NV1: Tenant Segregation (3.16), Cross-Platform Conformance (3.17), AI/ML Pipeline (3.18).
