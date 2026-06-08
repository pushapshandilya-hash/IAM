# Securing the CI/CD Pipeline in Azure

**A Security Architecture & Best-Practice Reference**

Source: GitHub → Build/Deploy: Azure DevOps Pipelines

| | |
|---|---|
| **Scope** | Azure DevOps Services (cloud) and Azure DevOps Server (self-hosted) |
| **Risk taxonomy** | OWASP Top 10 CI/CD Security Risks (v1.0) |
| **Control sources** | Microsoft Learn (official documentation) |
| **Maturity scale** | Azure Well-Architected Framework Security Maturity Model (Levels 1–5) |
| **Version** | 1.0 · Compiled June 2026 |

---

## Contents

1. [Overview & Scope](#1-overview--scope)
2. [Design Principles & Shared Responsibility](#2-design-principles--shared-responsibility)
3. [Reference Architecture & Solution Design](#3-reference-architecture--solution-design)
4. [Assets & Threat Model](#4-assets--threat-model)
5. [Defense-in-Depth Control Architecture](#5-defense-in-depth-control-architecture)
6. [Assurance & Verification](#6-assurance--verification)
7. [Maturity Model & Roadmap](#7-maturity-model--roadmap)
8. [Trade-offs & Residual-Risk Register](#8-trade-offs--residual-risk-register)
- [Appendix A — Azure Security Services Inventory](#appendix-a--azure-security-services-inventory)
- [Appendix B — Control-ID Traceability Matrix](#appendix-b--control-id-traceability-matrix)
- [Appendix C — OWASP CI/CD Risk Reference](#appendix-c--owasp-cicd-risk-reference)
- [Appendix D — Gap-Assessment Template](#appendix-d--gap-assessment-template)
- [Appendix E — Sources, Glossary & Document Control](#appendix-e--sources-glossary--document-control)

---

## 1. Overview & Scope

### 1.1 Executive Overview

Continuous integration and continuous delivery (CI/CD) pipelines are the path that moves code from a developer's workstation into production. That same path is now a primary target: a compromised pipeline gives an adversary a direct route to production systems, source code, signing material, and the secrets used to reach every connected environment. Microsoft's Well-Architected guidance treats the pipeline as part of the protected estate, **extending security scope across the full software supply chain — developer workstations, source repositories, build systems, and deployment environments** — rather than treating it as build tooling outside the security boundary.

This document defines a **secure-by-design target state** for an organization whose source code is hosted in GitHub and whose builds and deployments run on Azure DevOps Pipelines. It is written from a security-architecture perspective: controls are justified by trust boundaries and threats, mapped to the Azure services that enable them, and sequenced against a recognized maturity model. Every control traces to official Microsoft documentation; the risk taxonomy used to structure threats is the OWASP Top 10 CI/CD Security Risks.

### 1.2 Top Threats Addressed

The design is built to mitigate the CI/CD attack patterns catalogued by OWASP, with priority given to:

- **Poisoned pipeline execution** — untrusted code (especially from forks) running in the build environment.
- **Inadequate identity and access management** — over-privileged human and machine identities across SCM, build, and cloud.
- **Insufficient credential hygiene** — long-lived secrets stored in pipelines, logs, or configuration.
- **Insufficient flow control** — code reaching production without enforced review, branch protection, or approval gates.
- **Insecure system configuration & insufficient logging** — default settings and missing audit visibility that hide compromise.

### 1.3 Scope

**In scope:** the CI/CD platform and its controls — GitHub as source control feeding Azure DevOps Pipelines for build and deployment; identity and secret management for the pipeline; build agents; artifact handling; pipeline observability; and the Azure security services that enable these controls. Both Azure DevOps Services (cloud) and Azure DevOps Server (self-hosted) are covered, with deployment-specific differences flagged throughout.

**Out of scope:** runtime application security of the deployed workload beyond the deployment gate; full cloud-workload protection of production infrastructure; data-plane security of business applications; and organization-wide identity governance beyond what the pipeline consumes. These are adjacent programs that this pipeline design integrates with but does not define.

### 1.4 Assumptions

- Greenfield: the organization is establishing this environment new, so the target state is described directly rather than as a migration from a legacy configuration.
- Microsoft Entra ID is available as the identity provider for Azure DevOps.
- The organization can adopt the Azure-native security services referenced; license-dependent services are flagged so they can be planned for explicitly.
- A current-state gap assessment is provided as a template (Appendix D) for teams that already operate a pipeline and need to measure against this target.

---

## 2. Design Principles & Shared Responsibility

### 2.1 Design Principles

Principles are stated as actionable tenets, each translated into the concrete CI/CD mechanism that realizes it. They are drawn from the Azure Well-Architected Framework security pillar and Microsoft's Zero Trust guidance for DevOps.

| Principle | What it means here | Realized by |
|---|---|---|
| Verify explicitly (Zero Trust) | No implicit trust between pipeline stages, identities, or environments; authenticate and authorize every access. | Workload identity federation (OIDC); Microsoft Entra ID authentication; service-connection branch controls. |
| Least privilege | Human and machine identities receive only the access required, for only as long as required. | Project-scoped build identities; scoped service connections; just-in-time / Privileged Identity Management (PIM). |
| Assume breach | Design to contain blast radius when — not if — a component is compromised. | Project and agent-pool segmentation; isolation of production artifacts; ephemeral Microsoft-hosted agents. |
| Defense in depth | Layered, independent controls so no single failure is catastrophic. | Identity, network, pipeline, artifact, endpoint, runtime, and observability control layers (Section 5). |
| Secure development lifecycle (shift-left) | Security requirements and checks are designed in from the start and run early in the pipeline. | Threat modeling; PR-stage validation; code, secret, and dependency scanning. |
| Don't replicate native controls | Use platform-native security where it exists rather than rebuilding it, to maximize coverage and reduce maintenance. | Azure-native services (Key Vault, Defender for Cloud, Entra ID) consumed directly per Well-Architected guidance. |

### 2.2 Control-Framework Backbone

Controls in this document are organized by the OWASP Top 10 CI/CD Security Risks (the risk spine) and are intended to align with the broader Microsoft Well-Architected Framework security pillar and its Security Maturity Model. Organizations with regulatory obligations can additionally map each control to an external standard (for example, the NIST Secure Software Development Framework); that mapping is left as an organizational extension because it depends on the specific compliance regime and is outside the Microsoft-sourced scope of this reference.

### 2.3 Shared Responsibility & Division of Control

Azure DevOps Services is a cloud (PaaS) offering. Microsoft secures and maintains the underlying platform and infrastructure; the customer is responsible for configuring security for their own organizations, projects, repositories, and pipelines. Establishing this division early prevents two failure modes: replicating controls Microsoft already provides, and assuming Microsoft covers controls that are in fact the customer's responsibility.

| Domain | Microsoft (platform) | Platform / DevOps team | Security team |
|---|---|---|---|
| Underlying infrastructure | Patching, availability, physical and platform security of Azure DevOps Services | — | Validate via Microsoft compliance attestations |
| Identity & access | Provides Entra ID, RBAC, service principals/managed identities | Configure RBAC, scope identities, disable inheritance | Define access policy, review privileged access |
| Pipelines & repos | Provides YAML engine, checks, branch policy features | Author secure pipelines, configure branch policies & approvals | Define required gates and review standards |
| Secrets | Provides Key Vault, workload identity federation | Integrate Key Vault, adopt OIDC, remove inline secrets | Define secret policy, rotation cadence, audit |
| Monitoring | Provides audit log, audit streaming, Defender for Cloud | Enable auditing, configure streaming to SIEM | Own detection use-cases and response |
| Self-hosted agents (Server / hybrid) | — (customer-operated) | Harden, patch, network-restrict, run low-privilege | Approve isolation model and network exposure |

> **Note:** on Azure DevOps Server (self-hosted), more of the "Microsoft (platform)" column shifts to the customer — notably infrastructure hardening, network controls, and (as Section 5 notes) audit streaming, which is not available on-premises.

---

## 3. Reference Architecture & Solution Design

### 3.1 Logical Reference Architecture

The pipeline is modeled as a sequence of components connected by data flows. Each component runs under an identity and sits within a trust zone; security controls are placed at the boundaries between zones, where trust changes.

| Flow # | Component | Identity context | Trust zone |
|---|---|---|---|
| F1 | Developer workstation | Developer user (Entra ID) | Untrusted-to-semi-trusted endpoint |
| F2 | GitHub repository (source) | Authenticated contributors; external fork contributors | Source-of-truth; mixed-trust (forks untrusted) |
| F3 | Azure Pipelines trigger / GitHub app | Azure Pipelines GitHub app (scoped) | Integration boundary |
| F4 | Build agent (Microsoft-hosted or self-hosted) | Build service identity; agent OS account | Execution zone — highest-value target |
| F5 | Secrets & service connections | Workload identity federation (OIDC) / managed identity | Credential boundary |
| F6 | Artifact (container image / package) | Build identity; registry identity | Integrity boundary |
| F7 | Deployment target (Azure / AKS) | Scoped service connection identity | Production zone |

### 3.2 Trust Boundaries & Control Points

Five boundaries carry the highest risk. Controls are concentrated where untrusted input or elevated privilege crosses a zone:

1. **Workstation → GitHub (F1→F2):** identity, MFA/SSO, and signed/reviewed commits; this is where supply-chain compromise frequently originates.
2. **GitHub → Pipeline (F2→F3→F4):** the fork/PR boundary — untrusted external code must not reach a privileged agent or receive secrets.
3. **Pipeline → Secrets (F4→F5):** the build must obtain credentials without long-lived secrets being exposed in YAML, logs, or fork builds.
4. **Build → Artifact (F4→F6):** only tested, scanned, trusted, version-pinned artifacts proceed.
5. **Artifact → Production (F6→F7):** deployment runs under a tightly scoped identity, gated by approvals and branch controls.

### 3.3 Solution Design — Secure-by-Default Pipeline Blueprint

The target design composes the layered controls of Section 5 into one opinionated, repeatable pattern rather than leaving each team to assemble controls ad hoc:

- **Pipeline as code, centrally governed:** YAML pipelines only (Classic pipeline creation disabled), with mandatory extends templates that define the enforced outer structure and required security steps so individual teams cannot remove them.
- **Enforced flow control:** branch policies require review before merge; approvals and checks (branch control, manual approval, optional business-hours) gate access to protected resources and environments.
- **Secretless authentication by default:** service connections use workload identity federation (OIDC); where a secret is unavoidable, it is sourced from Azure Key Vault via a variable group, never inline in YAML.
- **Isolated, ephemeral execution:** Microsoft-hosted agents provide a clean VM per run; where self-hosted agents are required (typically Server/hybrid), they are network-restricted, low-privilege, and segmented by project and by production vs. non-production.
- **Controlled promotion:** artifacts move dev → test → production only through environments protected by checks, with production artifacts isolated to a dedicated agent pool.
- **Observability wired in:** audit logging is enabled and (on Services) streamed to a SIEM; posture is monitored centrally via Defender for Cloud DevOps security.

### 3.4 Deployment-Model Design Fork: Services vs. Server

The two Azure DevOps deployment models diverge enough to change the design, not merely individual control settings:

| Design aspect | Azure DevOps Services (cloud) | Azure DevOps Server (self-hosted) |
|---|---|---|
| Build agents | Prefer Microsoft-hosted (isolated, ephemeral VM per run) | Self-hosted by necessity — must be hardened, patched, network-restricted, low-privilege, and segmented |
| Network controls | IP allowlisting and platform-managed network security | Customer owns network isolation: NSGs, restrictive firewalls, WAF where applicable |
| Audit streaming / SIEM | Native audit streaming to Azure Monitor Logs, Microsoft Sentinel, Splunk, or Event Grid | Native auditing/streaming not available; Splunk audit stream connection is the partial option |
| Platform patching | Microsoft-managed | Customer-managed — part of the security responsibility |

---

## 4. Assets & Threat Model

### 4.1 Asset Inventory (What We Protect)

Threat modeling is anchored to the assets a CI/CD compromise would expose. Each is rated by sensitivity to drive control priority.

| Asset | Why it matters | Sensitivity |
|---|---|---|
| Source code (GitHub) | Intellectual property; tampering injects malicious code downstream | High |
| Pipeline secrets / tokens | Direct access to connected cloud and third-party systems | Critical |
| Service-connection identities | Authenticate the pipeline to production Azure resources | Critical |
| Build agents | Code execution context; a compromised agent can exfiltrate everything in reach | Critical |
| Build artifacts / images | Tampering produces a trusted-looking malicious release | High |
| Signing / provenance material | Forged artifacts pass integrity checks if compromised | Critical |
| Audit logs | Loss or tampering hides compromise and breaks investigation | High |

### 4.2 Methodology & Threat Actors

The model uses an attack-path / STRIDE-style approach: for each trust boundary in Section 3.2, threats are identified, mapped to the relevant OWASP CI/CD risk, and rated. Threat actors considered:

- **External fork contributor:** submits a pull request whose code attempts to execute in the build environment or capture secrets (poisoned pipeline execution).
- **Compromised dependency / third-party service:** a malicious package or over-privileged integration runs code in the build (dependency-chain abuse, ungoverned third-party services).
- **Compromised developer endpoint:** stolen credentials or tokens from a workstation used to push malicious changes or access the pipeline.
- **Malicious or careless insider:** over-privileged identity used to alter pipelines, exfiltrate secrets, or push unreviewed code to production.

### 4.3 Threat-to-Boundary Mapping & Priority

Priority is derived from likelihood × impact and drives the control sequencing in Section 7. The OWASP column references the CI/CD risk identifier (CICD-SEC-n).

| Boundary | Representative threat | OWASP | Likelihood | Priority |
|---|---|---|---|---|
| F2→F4 (fork/PR) | Untrusted fork code executes on a privileged agent or reads secrets | CICD-SEC-4 | High | **Critical** |
| Identity (all) | Over-privileged human/machine identity enables lateral movement | CICD-SEC-2 | High | **Critical** |
| F4→F5 (secrets) | Long-lived secret leaked via YAML, logs, or fork build | CICD-SEC-6 | High | **Critical** |
| F1→F2 (commit) | Unreviewed/unsigned code reaches main and flows to production | CICD-SEC-1 | Medium | High |
| Config (platform) | Insecure defaults / Classic pipelines expose attack surface | CICD-SEC-7 | Medium | High |
| F3 (3rd party) | Marketplace task or over-scoped GitHub app abused | CICD-SEC-8 | Medium | High |
| F4→F6 (artifact) | Tampered or vulnerable image promoted as trusted | CICD-SEC-9 | Medium | High |
| Observability | Compromise undetected due to missing/short-retention logs | CICD-SEC-10 | Medium | High |

---

## 5. Defense-in-Depth Control Architecture

Controls are organized into independent layers so that a failure in one does not collapse the others. Each control has a stable identifier (used throughout this document and in the gap-assessment template), a rationale, the enabling Azure service, the OWASP risk it addresses, deployment scope, and a maturity tier (**L1** = baseline / Well-Architected Level 1–2; **L3** = intermediate; **L4+** = advanced). License-dependent controls are marked 🔶. Detection signals and failure modes for each control are consolidated in Section 6.

### 5.1 Identity Plane

Identity is the primary control plane; least privilege here limits blast radius everywhere else.

| ID | Control & rationale | Enabling Azure service | OWASP / Scope / Tier |
|---|---|---|---|
| CTRL-IAM-01 | Use Microsoft Entra ID as the identity plane and apply least-privilege RBAC to all human and machine access. | Microsoft Entra ID | CICD-SEC-2 · Both · L1 |
| CTRL-IAM-02 | Use project-scoped build identities, not collection-level, so a pipeline can only reach its own project's resources. | Azure Pipelines build identity (project scope) | CICD-SEC-2/5 · Both · L1 |
| CTRL-IAM-03 | Manage security at org/project/object level and disable permission inheritance where possible. | Azure DevOps permissions & security groups | CICD-SEC-2 · Both · L3 |
| CTRL-IAM-04 | Limit user visibility and collaboration to specific projects for restricted users/guests. | Project-scoped Users group | CICD-SEC-2 · Both · L3 |
| CTRL-IAM-05 | Use managed identities / service principals for non-human authentication instead of stored credentials. | Microsoft Entra managed identities / service principals | CICD-SEC-6 · Both · L1 |
| CTRL-IAM-06 | Apply just-in-time elevation and Privileged Identity Management for privileged/break-glass access. | Microsoft Entra PIM (JIT) | CICD-SEC-2 · Both · L4+ |

### 5.2 Network & Isolation

Reduces exposure of the platform and contains self-hosted execution. Weighted more heavily for Server/hybrid.

| ID | Control & rationale | Enabling Azure service | OWASP / Scope / Tier |
|---|---|---|---|
| CTRL-NET-01 | Restrict access with IP allowlisting so only trusted sources reach the organization. | Azure DevOps IP allowlisting | CICD-SEC-7 · Both · L3 |
| CTRL-NET-02 | Encrypt data in transit and at rest; secure channels with HTTPS. | Azure encryption | CICD-SEC-7 · Both · L1 |
| CTRL-NET-03 | Place a Web Application Firewall in front of exposed surfaces to filter malicious traffic. | Azure WAF (Front Door / App Gateway) | CICD-SEC-7 · Server/hybrid · L4+ |
| CTRL-NET-04 | Use network security groups to control inbound/outbound traffic to Azure resources. | Network security groups (NSGs) | CICD-SEC-7 · Server/hybrid · L3 |
| CTRL-NET-05 | Configure restrictive firewalls for self-hosted agents — only required endpoints reachable. | Self-hosted agent firewall config | CICD-SEC-7 · Server/hybrid · L3 |

### 5.3 Pipeline Integrity & Flow Control

Prevents untrusted code from executing and ensures only reviewed, gated changes proceed. This layer carries the highest-priority threats.

| ID | Control & rationale | Enabling Azure service | OWASP / Scope / Tier |
|---|---|---|---|
| CTRL-PIPE-01 | Use YAML pipelines (infrastructure as code) and disable creation of Classic pipelines. | Azure Pipelines (YAML); disable Classic | CICD-SEC-1/7 · Both · L1 |
| CTRL-PIPE-02 | Apply branch policies and repository checks so code and pipeline changes are reviewed. | Branch policies; repository resource checks | CICD-SEC-1 · Both · L1 |
| CTRL-PIPE-03 | Protect fork builds: do not expose secrets to forks, trigger fork builds manually, run them on Microsoft-hosted agents, and use the Azure Pipelines GitHub app to limit token scope. | Fork-build settings; Azure Pipelines GitHub app | CICD-SEC-4 · Both · L1 |
| CTRL-PIPE-04 | Gate protected resources/environments with approvals and checks (branch control, manual approval, business hours). | Approvals and checks | CICD-SEC-1/5 · Both · L3 |
| CTRL-PIPE-05 | Enforce a secure outer structure with extends templates teams cannot bypass. | Pipeline extends templates | CICD-SEC-1/4 · Both · L3 |
| CTRL-PIPE-06 | Govern third-party code: disable installing/running Marketplace tasks; scope the GitHub app to intended repos only. | Marketplace task control; GitHub app scoping | CICD-SEC-8 · Both · L3 |
| CTRL-PIPE-07 | Prevent malicious execution: validate inputs, use runtime parameters, avoid PATH reliance, enable shell-argument validation. | Runtime parameters; shell-arg validation setting | CICD-SEC-4 · Both · L3 |

### 5.4 Secrets & Credentials

The strongest secret is no secret. Eliminate long-lived credentials; where unavoidable, vault and rotate them.

| ID | Control & rationale | Enabling Azure service | OWASP / Scope / Tier |
|---|---|---|---|
| CTRL-SEC-01 | Authenticate service connections with workload identity federation (OIDC) instead of stored secrets. | Workload identity federation (OIDC) | CICD-SEC-6 · Both · L1 |
| CTRL-SEC-02 | Source any required secret from Azure Key Vault via a variable group; minimize service-connection scope to a specific resource group. | Azure Key Vault; scoped service connections | CICD-SEC-6 · Both · L1 |
| CTRL-SEC-03 | Never store secrets in YAML; do not log or print secrets; do not use structured data as a single secret. | Pipeline secret-handling settings | CICD-SEC-6 · Both · L1 |
| CTRL-SEC-04 | Limit queue-time variables and constrain settable variables to prevent injection of new variables. | Queue-time variable limit; settableVariables | CICD-SEC-4/6 · Both · L3 |
| CTRL-SEC-05 | Audit secret handling in tasks and logs; review and remove unused secrets; rotate regularly. | Secret audit & rotation process | CICD-SEC-6 · Both · L3 |

### 5.5 Artifact Integrity & Provenance

Ensures only tested, trusted artifacts are promoted. **Coverage note:** Azure-native artifact vulnerability scanning and trusted-image controls are well-supported; however, full build-provenance / cryptographic attestation (OWASP CICD-SEC-9) has thinner first-party Microsoft coverage than vulnerability scanning. This is recorded as a residual gap in Section 8 (RR-03) rather than overstated.

| ID | Control & rationale | Enabling Azure service | OWASP / Scope / Tier |
|---|---|---|---|
| CTRL-ART-01 | Use trusted, verified, version-pinned images (never the latest tag); update base images for patches. | Azure Container Registry (trusted images) | CICD-SEC-9 · Both · L3 |
| CTRL-ART-02 🔶 | Scan artifacts/images for vulnerabilities before deployment. | ACR integrated scanning; Microsoft Defender for Cloud | CICD-SEC-3/9 · Both · L3 |
| CTRL-ART-03 | Establish artifact provenance/integrity validation for promoted artifacts (partial native coverage — see RR-03). | Defender for Cloud insights; organizational attestation process | CICD-SEC-9 · Both · L4+ |

### 5.6 Developer Endpoint

Microsoft's secure-development guidance explicitly extends scope to developer workstations, a common origin of supply-chain compromise.

| ID | Control & rationale | Enabling Azure service | OWASP / Scope / Tier |
|---|---|---|---|
| CTRL-EP-01 | Use hardened secure admin workstations (SAWs) for changes to high-risk and production environments. | Secure admin workstations (Zero Trust endpoints) | CICD-SEC-2 · Both · L4+ |
| CTRL-EP-02 | Secure the developer environment: least privilege, branch security, and only trusted tools, extensions, and integrations. | Zero Trust developer-environment guidance | CICD-SEC-3/8 · Both · L3 |

### 5.7 Runtime (Container / AKS)

Where pipelines deploy containers, runtime controls limit what a compromised workload can do.

| ID | Control & rationale | Enabling Azure service | OWASP / Scope / Tier |
|---|---|---|---|
| CTRL-RT-01 | Enforce least-privilege containers: AKS RBAC, Pod Security Admission, non-root execution. | AKS RBAC; Pod Security Admission | CICD-SEC-7 · Both · L4+ |
| CTRL-RT-02 | Restrict container communication and enforce trusted images via policy. | Network Policies; Azure Policy for AKS | CICD-SEC-7/9 · Both · L4+ |
| CTRL-RT-03 | Mark task/tool volumes read-only and set CPU/memory resource limits. | Container/agent configuration | CICD-SEC-7 · Both · L3 |
| CTRL-RT-04 🔶 | Monitor and detect runtime threats and image risks. | Microsoft Defender for Cloud / Defender for Containers | CICD-SEC-7 · Both · L4+ |

### 5.8 Code & Dependency Scanning

| ID | Control & rationale | Enabling Azure service | OWASP / Scope / Tier |
|---|---|---|---|
| CTRL-SCAN-01 🔶 | Automate secret, code, and dependency scanning across repositories. | GitHub Advanced Security for Azure DevOps | CICD-SEC-3/6 · Both · L3 |

> 🔶 Flagged: GitHub Advanced Security for Azure DevOps is a billed add-on (activated via "Begin billing").

### 5.9 Observability

Visibility is the control that lets every other control be verified. Note the Services-vs-Server divergence.

| ID | Control & rationale | Enabling Azure service | OWASP / Scope / Tier |
|---|---|---|---|
| CTRL-OBS-01 | Enable audit logging (Log Audit Events) for the organization. | Azure DevOps Auditing | CICD-SEC-10 · Services · L1 |
| CTRL-OBS-02 🔶 | Stream audit events to a SIEM for detection and retention. | Audit streaming → Azure Monitor Logs / Microsoft Sentinel / Splunk / Event Grid | CICD-SEC-10 · Services (Splunk partial on Server) · L3 |
| CTRL-OBS-03 🔶 | Monitor DevOps security posture centrally across connected environments. | Microsoft Defender for Cloud — DevOps security | CICD-SEC-10 · Both · L4+ |
| CTRL-OBS-04 🔶 | Configure log retention beyond the 30-day default to meet investigation/compliance needs. | Azure Monitor Logs retention settings | CICD-SEC-10 · Services · L3 |

---

## 6. Assurance & Verification

A control that cannot be verified cannot be relied upon. This section distinguishes preventive controls (stop the action) from detective controls (reveal it), gives the detection signal and failure mode for each control area, and defines continuous validation. Because this document carries no separate metrics section, the signals below double as the measures of control effectiveness.

### 6.1 Continuous Validation

- **Posture drift:** Microsoft Defender for Cloud DevOps security surfaces configuration issues across connected environments, providing ongoing rather than point-in-time assurance.
- **Policy enforcement:** Azure Policy (including Azure Policy for AKS) enforces required configurations automatically, so compliance is continuous rather than audited after the fact.
- **Audit evidence:** the `AzureDevOpsAuditing` table (via audit streaming) provides queryable evidence for investigation and review; retention must be configured beyond the 30-day default to preserve it.

### 6.2 Detection Signal & Failure Mode by Control Area

| Control area | Type | Detection signal | Failure mode if bypassed |
|---|---|---|---|
| Identity (IAM) | Preventive | Audit events for permission/role changes; privileged-access (PIM) activation logs | Over-privileged identity enables lateral movement across projects and to production |
| Network (NET) | Preventive | Denied-connection logs; NSG flow logs (Server/hybrid) | Platform or self-hosted agent reachable from untrusted networks |
| Pipeline integrity (PIPE) | Preventive | Pipeline/branch-policy change audit events; fork-build trigger records | Untrusted fork code executes on a privileged agent; unreviewed code reaches production |
| Secrets (SEC) | Preventive + detective | Secret-access patterns; scrubbed-log verification; rotation tracking | Long-lived secret leaks and grants standing access to connected systems |
| Artifact (ART) | Detective | Vulnerability scan results; image provenance/insight findings | Tampered or vulnerable artifact promoted as trusted |
| Endpoint (EP) | Preventive | Workstation compliance posture; privileged-action origin | Credential theft from a workstation used to push malicious change |
| Runtime (RT) | Detective | Defender for Containers alerts; policy-violation events | Compromised container escalates privilege or moves laterally |
| Observability (OBS) | Detective | Presence and completeness of streamed audit events; alert coverage | Compromise proceeds undetected; investigation impossible after retention lapses |

---

## 7. Maturity Model & Roadmap

Maturity is measured against the Azure Well-Architected Framework Security Maturity Model. A greenfield organization starts below Level 1; the roadmap sequences controls to reach a defensible baseline first, then intermediate and advanced posture. Sequencing respects control dependencies (for example, Entra ID integration precedes RBAC scoping; a Log Analytics workspace precedes Microsoft Sentinel).

### 7.1 Maturity Tiers & Exit Criteria

| Tier | Posture | Exit criteria (representative) |
|---|---|---|
| L1 — Baseline | Minimum secure foundation, enabled before workloads ship | Entra ID identity; project-scoped identities; YAML-only; branch policies; fork-build protections; OIDC/Key Vault secrets; no secrets in YAML; auditing enabled |
| L3 — Intermediate | Hardened configuration and enforced governance | Disabled permission inheritance; approvals & checks; extends templates; Marketplace/third-party governance; scanning; audit streaming to SIEM; secret rotation |
| L4+ — Advanced | Proactive, continuously validated, contained | JIT/PIM; SAWs; runtime (AKS) controls; Defender for Cloud posture management; artifact provenance; WAF where applicable |

### 7.2 Phased Roadmap

**Day 0 — Minimum secure baseline (non-negotiable before first production pipeline):** CTRL-IAM-01/02/05, CTRL-PIPE-01/02/03, CTRL-SEC-01/02/03, CTRL-OBS-01, CTRL-NET-02. These close the Critical-priority threats from Section 4.3.

**Phase 1 — Reach L1 baseline (weeks):** complete all baseline controls; verify fork protections and secretless authentication end to end; confirm auditing is on.

**Phase 2 — Reach L3 intermediate (1–2 quarters):** disable permission inheritance; add approvals/checks and extends templates; govern Marketplace tasks and the GitHub app; enable scanning (plan GitHub Advanced Security spend); stand up a Log Analytics workspace and stream audit events to Microsoft Sentinel; formalize secret rotation.

**Phase 3 — Reach L4+ advanced (ongoing):** introduce JIT/PIM and SAWs for privileged paths; apply AKS runtime controls; enable Defender for Cloud DevOps posture management; address artifact provenance (RR-03); add WAF/NSG hardening for Server/hybrid.

> **Dependency notes:** enable Microsoft Entra ID integration before scoping RBAC (CTRL-IAM-03/04); create the Log Analytics workspace before connecting Microsoft Sentinel (CTRL-OBS-02); audit streaming and its retention apply to Azure DevOps Services — on Server, plan a Splunk-based alternative.

---

## 8. Trade-offs & Residual-Risk Register

Microsoft's Well-Architected guidance directs architects to document and formalize security trade-offs. This register records accepted residual risk, each traced to a threat from Section 4, with a treatment and a residual rating. It is a living artifact: owners review it as posture and tooling change.

### 8.1 Security vs. Developer Velocity

The central tension in CI/CD security is friction versus speed. The design manages it deliberately: controls are made the default path (extends templates, secretless auth, Microsoft-hosted agents) so the secure way is also the easy way, and the heaviest gates (manual fork triggers, approvals, JIT) are concentrated at the highest-risk boundaries rather than applied uniformly. Where a control materially slows delivery, the trade-off is recorded below rather than silently dropped.

### 8.2 Residual-Risk Register

| Risk ID | Description | Linked threat | Treatment | Residual rating |
|---|---|---|---|---|
| RR-01 | Self-hosted agents (required on Server/hybrid) expand attack surface vs. ephemeral Microsoft-hosted agents | CICD-SEC-4/7 (F4) | Harden, patch, network-restrict, run low-privilege, segment by project & prod/non-prod | Medium |
| RR-02 | Default audit-log retention is 30 days; investigations may need longer history | CICD-SEC-10 | Configure extended retention (incurs cost); stream to SIEM for long-term storage | Low |
| RR-03 | Full artifact build-provenance/attestation has thinner first-party Microsoft coverage | CICD-SEC-9 (F6) | Use ACR scanning + trusted, pinned images now; track native provenance maturity; supplement with organizational attestation | Medium |
| RR-04 | Audit streaming/Sentinel detection is unavailable natively on Azure DevOps Server | CICD-SEC-10 | Use Splunk audit-stream connection on Server; weight other detective controls | Medium |
| RR-05 | Advanced scanning and posture management are license-dependent; cost may delay adoption | CICD-SEC-3/6/10 | Sequence in Phase 2–3; until enabled, rely on baseline preventive controls | Medium |

---

## Appendix A — Azure Security Services Inventory

Consolidated view of the Azure services that enable the controls, with prerequisites, dependencies, and cost signals. License-dependent items are flagged 🔶.

| Service | Enables (control area) | Dependencies / prerequisites | Cost / license |
|---|---|---|---|
| Microsoft Entra ID | Identity, RBAC, managed identities, PIM (IAM) | Entra ID tenant connected to Azure DevOps | Included; PIM may require premium |
| Workload identity federation (OIDC) | Secretless service-connection auth (SEC) | Entra ID; service connection | Included |
| Azure Key Vault | Secret storage via variable groups (SEC) | Key Vault provisioned; access policy/RBAC | Low (per-operation) |
| Microsoft-hosted agents | Isolated, ephemeral build execution (PIPE/NET) | Azure DevOps Services | Included / parallelism-based |
| GitHub Advanced Security for Azure DevOps 🔶 | Secret/code/dependency scanning (SCAN) | Enable per repo/org; .NET runtime on self-hosted agents | Billed (per active committer) |
| Azure Container Registry | Trusted images + integrated scanning (ART) | ACR provisioned | Tiered |
| Microsoft Defender for Cloud (DevOps security / Containers) 🔶 | Posture management; runtime threat detection (OBS/RT/ART) | Connect Azure DevOps/GitHub orgs | Plan-dependent (billed) |
| Azure DevOps Auditing + streaming 🔶 | Audit log and SIEM export (OBS) | Services only; Log Analytics workspace for Sentinel | Ingestion/retention billed; 30-day default |
| Microsoft Sentinel 🔶 | SIEM/SOAR detection & response (OBS) | Log Analytics workspace; audit stream connected | Billed (ingestion) |
| Azure Policy / Policy for AKS | Configuration & container policy enforcement (RT) | AKS for container policies | Included (Policy) |
| NSGs / Azure WAF | Network isolation & filtering (NET) | Relevant for Server/hybrid networking | WAF billed; NSGs included |

---

## Appendix B — Control-ID Traceability Matrix

The through-line linking each control to its OWASP CI/CD risk, enabling service, deployment scope, and maturity tier.

| Control ID | Control | OWASP | Scope | Tier |
|---|---|---|---|---|
| CTRL-IAM-01 | Entra ID identity + least-privilege RBAC | CICD-SEC-2 | Both | L1 |
| CTRL-IAM-02 | Project-scoped build identities | CICD-SEC-2/5 | Both | L1 |
| CTRL-IAM-03 | Manage at org/project/object; disable inheritance | CICD-SEC-2 | Both | L3 |
| CTRL-IAM-04 | Limit user/project visibility | CICD-SEC-2 | Both | L3 |
| CTRL-IAM-05 | Managed identities / service principals | CICD-SEC-6 | Both | L1 |
| CTRL-IAM-06 | JIT / PIM privileged access | CICD-SEC-2 | Both | L4+ |
| CTRL-NET-01 | IP allowlisting | CICD-SEC-7 | Both | L3 |
| CTRL-NET-02 | Encryption in transit & at rest | CICD-SEC-7 | Both | L1 |
| CTRL-NET-03 | Web Application Firewall | CICD-SEC-7 | Server/hybrid | L4+ |
| CTRL-NET-04 | Network security groups | CICD-SEC-7 | Server/hybrid | L3 |
| CTRL-NET-05 | Self-hosted agent firewalls | CICD-SEC-7 | Server/hybrid | L3 |
| CTRL-PIPE-01 | YAML only; disable Classic | CICD-SEC-1/7 | Both | L1 |
| CTRL-PIPE-02 | Branch policies & repo checks | CICD-SEC-1 | Both | L1 |
| CTRL-PIPE-03 | Fork-build protections + GitHub app scoping | CICD-SEC-4 | Both | L1 |
| CTRL-PIPE-04 | Approvals and checks | CICD-SEC-1/5 | Both | L3 |
| CTRL-PIPE-05 | Extends templates | CICD-SEC-1/4 | Both | L3 |
| CTRL-PIPE-06 | Third-party / Marketplace governance | CICD-SEC-8 | Both | L3 |
| CTRL-PIPE-07 | Input validation & shell-arg validation | CICD-SEC-4 | Both | L3 |
| CTRL-SEC-01 | Workload identity federation (OIDC) | CICD-SEC-6 | Both | L1 |
| CTRL-SEC-02 | Key Vault + scoped service connections | CICD-SEC-6 | Both | L1 |
| CTRL-SEC-03 | No secrets in YAML; no secret logging | CICD-SEC-6 | Both | L1 |
| CTRL-SEC-04 | Limit queue-time / settable variables | CICD-SEC-4/6 | Both | L3 |
| CTRL-SEC-05 | Audit & rotate secrets | CICD-SEC-6 | Both | L3 |
| CTRL-ART-01 | Trusted, pinned images | CICD-SEC-9 | Both | L3 |
| CTRL-ART-02 | Artifact/image vulnerability scanning | CICD-SEC-3/9 | Both | L3 |
| CTRL-ART-03 | Artifact provenance/integrity (partial) | CICD-SEC-9 | Both | L4+ |
| CTRL-EP-01 | Secure admin workstations (SAWs) | CICD-SEC-2 | Both | L4+ |
| CTRL-EP-02 | Secure developer environment | CICD-SEC-3/8 | Both | L3 |
| CTRL-RT-01 | AKS RBAC / Pod Security Admission | CICD-SEC-7 | Both | L4+ |
| CTRL-RT-02 | Network Policies / Azure Policy for AKS | CICD-SEC-7/9 | Both | L4+ |
| CTRL-RT-03 | Read-only volumes; resource limits | CICD-SEC-7 | Both | L3 |
| CTRL-RT-04 | Defender for Containers runtime detection | CICD-SEC-7 | Both | L4+ |
| CTRL-SCAN-01 | GitHub Advanced Security scanning | CICD-SEC-3/6 | Both | L3 |
| CTRL-OBS-01 | Enable auditing | CICD-SEC-10 | Services | L1 |
| CTRL-OBS-02 | Audit streaming to SIEM | CICD-SEC-10 | Services / Server (Splunk) | L3 |
| CTRL-OBS-03 | Defender for Cloud posture | CICD-SEC-10 | Both | L4+ |
| CTRL-OBS-04 | Extended log retention | CICD-SEC-10 | Services | L3 |

---

## Appendix C — OWASP CI/CD Risk Reference

The risk spine used to structure the threat model and control mapping (OWASP Top 10 CI/CD Security Risks, v1.0).

| ID | Risk | Primary mitigating control areas |
|---|---|---|
| CICD-SEC-1 | Insufficient Flow Control Mechanisms | Branch policies, approvals & checks, extends templates |
| CICD-SEC-2 | Inadequate Identity and Access Management | Entra ID RBAC, scoped identities, PIM, SAWs |
| CICD-SEC-3 | Dependency Chain Abuse | Dependency scanning, trusted developer environment |
| CICD-SEC-4 | Poisoned Pipeline Execution | Fork-build protections, input validation, templates |
| CICD-SEC-5 | Insufficient PBAC (Pipeline-Based Access Controls) | Project-scoped identities, approvals & checks |
| CICD-SEC-6 | Insufficient Credential Hygiene | OIDC, Key Vault, no secrets in YAML, rotation |
| CICD-SEC-7 | Insecure System Configuration | YAML-only, network controls, agent hardening, runtime policy |
| CICD-SEC-8 | Ungoverned Usage of 3rd Party Services | Marketplace task control, GitHub app scoping |
| CICD-SEC-9 | Improper Artifact Integrity Validation | Trusted/pinned images, scanning, provenance (partial) |
| CICD-SEC-10 | Insufficient Logging and Visibility | Auditing, streaming to SIEM, Defender posture, retention |

---

## Appendix D — Gap-Assessment Template

For teams with an existing environment: score current state per control against the target tier on the Well-Architected scale (Level 0 = not implemented, 1 = baseline, 3 = intermediate, 5 = advanced/optimized). Record the gap and an owner. Extend with the full control set from Appendix B.

| Control ID | Control | Target tier | Current (0–5) | Gap / action / owner |
|---|---|---|---|---|
| CTRL-IAM-01 | Entra ID + least-privilege RBAC | L1 | | |
| CTRL-PIPE-03 | Fork-build protections | L1 | | |
| CTRL-SEC-01 | Workload identity federation (OIDC) | L1 | | |
| CTRL-OBS-01 | Enable auditing | L1 | | |
| CTRL-PIPE-04 | Approvals and checks | L3 | | |
| CTRL-SCAN-01 | Advanced scanning | L3 | | |
| CTRL-OBS-02 | Audit streaming to SIEM | L3 | | |
| CTRL-IAM-06 | JIT / PIM | L4+ | | |
| CTRL-ART-03 | Artifact provenance | L4+ | | |
| CTRL-OBS-03 | Defender for Cloud posture | L4+ | | |

---

## Appendix E — Sources, Glossary & Document Control

### Microsoft Learn sources (control basis)

All accessed June 2026.

- [Secure your Azure Pipelines](https://learn.microsoft.com/en-us/azure/devops/pipelines/security/overview)
- [Make your Azure DevOps secure](https://learn.microsoft.com/en-us/azure/devops/organizations/security/security-overview)
- [DevOps security considerations (Cloud Adoption Framework)](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/ready/considerations/security-considerations-overview)
- [Secure DevOps environments for Zero Trust](https://learn.microsoft.com/en-us/azure/cloud-adoption-framework/secure/development-security-implementation-operations)
- [Securing a development lifecycle (Well-Architected Framework)](https://learn.microsoft.com/en-us/azure/well-architected/security/secure-development-lifecycle)
- [Security Maturity Model (Well-Architected Framework)](https://learn.microsoft.com/en-us/azure/well-architected/security/maturity-model)
- [Defender for Cloud DevOps security](https://learn.microsoft.com/en-us/azure/defender-for-cloud/defender-for-devops-introduction)
- [GitHub Advanced Security for Azure DevOps](https://learn.microsoft.com/en-us/azure/devops/repos/security/configure-github-advanced-security-features)
- [Authentication, authorization & security policies](https://learn.microsoft.com/en-us/azure/devops/organizations/security/about-security-identity)
- [Permissions and security groups](https://learn.microsoft.com/en-us/azure/devops/organizations/security/about-permissions)
- [Data protection overview (Azure DevOps Services)](https://learn.microsoft.com/en-us/azure/devops/organizations/security/data-protection)
- [Create audit streaming for Azure DevOps](https://learn.microsoft.com/en-us/azure/devops/organizations/audit/auditing-streaming)

### Risk taxonomy reference

- [OWASP Top 10 CI/CD Security Risks](https://owasp.org/www-project-top-10-ci-cd-security-risks/) (v1.0, accessed June 2026) — used solely as the risk-mapping taxonomy.

### Glossary

| Term | Meaning |
|---|---|
| OIDC / Workload identity federation | OpenID Connect-based authentication between Azure and Azure DevOps without storing long-lived secrets |
| PIM | Privileged Identity Management — just-in-time elevation of privileged roles |
| SAW | Secure Admin Workstation — hardened device for high-risk/production changes |
| GHAS for Azure DevOps | GitHub Advanced Security for Azure DevOps — secret/code/dependency scanning (billed) |
| SIEM / SOAR | Security Information and Event Management / Security Orchestration, Automation and Response |
| SLSA-style provenance | Verifiable build-origin metadata for artifacts (referenced as a partial-coverage area, RR-03) |
| CICD-SEC-n | Identifier in the OWASP Top 10 CI/CD Security Risks list |

### Document control

| Field | Value |
|---|---|
| Version | 1.0 |
| Status | Reference / target-state design |
| Scope | CI/CD security — GitHub source + Azure DevOps Pipelines; Services & Server |
| Control source | Microsoft Learn (official) |
| Risk taxonomy | OWASP Top 10 CI/CD Security Risks v1.0 |
| Review cadence | Living document — review on tooling/posture change or at least annually |
