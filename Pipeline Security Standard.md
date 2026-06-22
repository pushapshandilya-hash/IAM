# CI/CD Pipeline Security Standard
### for Multimodel, Multitenant, and Agentic AI Environments

> A controls-based standard for securing continuous integration and continuous delivery pipelines in which multiple AI models operate, infrastructure is shared across tenants, and autonomous AI agents hold write and execute authority within the pipeline.

| | |
|---|---|
| **Owner** | Office of the Security Architect / DevSecOps Governance |
| **Classification** | Internal |
| **Version** | 2.0 (Draft) |
| **Status** | For Review |
| **Date** | June 2026 |

This is a normative control document. The keywords **MUST**, **MUST NOT**, **SHOULD**, and **MAY** follow RFC 2119. A **MUST** control is mandatory for every in-scope pipeline at its applicable tier; deviations require a time-bound exception approved under [Section 13](#13-governance-enforcement-and-exceptions).

---

## Contents

1. [Executive Summary and Purpose](#1-executive-summary-and-purpose)
2. [Minimum Production Baseline](#2-minimum-production-baseline)
3. [CI/CD Security Risks and Why They Matter](#3-cicd-security-risks-and-why-they-matter)
4. [Components and Their Controls](#4-components-and-their-controls)
5. [Multimodel and Multitenant Challenges](#5-multimodel-and-multitenant-challenges)
6. [Types of CI/CD Pipelines Covered](#6-types-of-cicd-pipelines-covered)
7. [A New Class of Risk: Agentic AI in CI/CD](#7-a-new-class-of-risk-agentic-ai-in-cicd)
8. [Security Controls by Pipeline Stage](#8-security-controls-by-pipeline-stage)
9. [Security Controls by Domain](#9-security-controls-by-domain)
10. [Secure Agentic Pipeline: Reference Architecture](#10-secure-agentic-cicd-pipeline-reference-architecture)
11. [Responsibilities (RACI)](#11-responsibilities-raci)
12. [Measuring Effectiveness](#12-measuring-effectiveness)
13. [Governance, Enforcement, and Exceptions](#13-governance-enforcement-and-exceptions)
- [Appendix A: Framework Control Mapping](#appendix-a-framework-control-mapping)
- [Appendix B: Glossary](#appendix-b-glossary)
- [Appendix C: References and Standards](#appendix-c-references-and-standards)

### Revision history

| Version | Date | Summary of change |
|---|---|---|
| 1.0 | Jun 2026 | Initial issue; agentic AI introduced as a first-class threat class. |
| 1.1 | Jun 2026 | Added Minimum Production Baseline, risk tiers, Governance section, per-control checks, and several enhanced controls. |
| 2.0 | Jun 2026 | Architecture review applied: corrected NIST mappings (SA-12 retired to the SR family in Rev 5, plus four loose mappings); resolved the Rule-of-Two consistency issue; added data/model-integrity and log-hygiene controls (LLM04, LLM02); softened the exception stance; consolidated lifecycle stages into a mapping table and merged the two component tables; added NIST AI RMF and SP 800-218A references; updated the Defender product name. |

### Normative references

Controls derive from, and should be read alongside, the frameworks below; inline identifiers (for example `SR-4` or `CICD-SEC-4`) refer to them.

- **NIST SP 800-53 Rev. 5** — Security and Privacy Controls (AC, AU, CM, IA, SA, SI, SC, SR, IR, CP).
- **NIST SP 800-218 (SSDF v1.1)** and **SP 800-218A** — secure software development, incl. the generative-AI extension.
- **NIST AI RMF (AI 100-1)** — AI Risk Management Framework.
- **OWASP Top 10 CI/CD Security Risks** (CICD-SEC-1..10) and **OWASP Top 10 for LLM Applications 2025** (LLM01..10).
- **Microsoft / Azure** — Microsoft Cloud Security Benchmark, Entra ID, Key Vault, Defender for Cloud (DevOps security), Sentinel.
- **NIST SP 800-207** (Zero Trust), **NIST SP 800-161** and **SLSA** (supply chain).

---

## 1. Executive Summary and Purpose

### 1.1 Purpose
This standard defines the security controls for CI/CD pipelines operated by the organization, so that software, model, and infrastructure changes reach production through a pipeline that is governed, auditable, and resistant to compromise — without controls so heavy that engineering velocity collapses. The pipeline is treated as production infrastructure: high-privilege, high-velocity, and a direct path to the organization's most sensitive systems.

### 1.2 Why this standard is needed now
Three forces have changed the risk picture. Pipelines now orchestrate multiple AI models, widening the attack surface beyond code to prompts, model outputs, and model supply chains. Infrastructure is increasingly multitenant, so a failure in one tenant's pipeline can leak into another's. And autonomous AI agents now hold write and execute authority inside pipelines, reading untrusted input while holding credentials and tools — a combination traditional CI/CD controls were not built to contain.

> **Why agentic AI changes the model.** In June 2026, Microsoft Threat Intelligence disclosed a prompt-injection pathway in an AI coding agent running as a GitHub Action: attacker-controlled text in an issue or PR body — invisible when rendered, readable by the model — could steer the agent into reading process environment files and exfiltrating a workflow credential, defeating both the model's refusal layer and the platform's secret scanner. The lesson generalizes: when a component treats natural language as instruction, untrusted input becomes executable, and the agent's prompt, tool permissions, and runtime isolation all become part of the security perimeter.

### 1.3 Scope
Applies to all pipelines that build, test, package, or deploy software or AI models into any environment the organization owns or operates, including pipelines that use AI models or agents at any stage. Covers the pipeline systems, the identities and secrets flowing through them, and the AI components that participate.

**Out of scope:** foundation-model safety alignment, end-user product security unrelated to delivery, and physical data-center security — each governed separately.

### 1.4 Intended audience

| Audience | How they use this standard |
|---|---|
| Platform / DevOps Engineering | Implements and operates controls in the platform; owns runners, identity wiring, isolation, and recovery. |
| DevSecOps / Application Security | Embeds scanning and policy gates; enforces the baseline as policy-as-code; audits compliance; manages exceptions. |
| AI / ML Engineering | Applies the agentic-AI, data-integrity, and model-supply-chain controls (Sections 7 and 9). |
| Tenant Operators | Configure tenant-scoped pipelines within the isolation boundaries defined here. |
| CISO / Governance | Owns the standard, approves exceptions (Section 13), and reviews effectiveness metrics (Section 12). |

### 1.5 How to read this standard
- **Tiered.** Controls scale with risk. Section 2 defines three tiers; the Minimum Production Baseline is the floor that gates every production deployment.
- **Verifiable.** Each MUST control carries an objective acceptance **Check** stating how conformance is confirmed, so it can be audited and, where possible, enforced automatically.
- **Stable IDs.** Control identifiers (`IAM-2`, `SCR-1`, `AGT-5.1`, …) are tied to a control domain, not a section number, so they survive revisions.
- **Enforced, not attested.** The baseline is enforced as policy-as-code (Section 13); detective-only checks are insufficient for it.
- **Single source of truth.** Each control is defined once, in Section 7 (agentic) or Section 9 (domains). Sections 2 and 8 reference those controls rather than restating them, and the framework crosswalk lives in Appendix A.

---

## 2. Minimum Production Baseline

> **The non-negotiable floor for production.**

Everything else in this standard is depth behind this floor. **No pipeline may deploy to production unless every baseline control below is met and evidenced.** The baseline is deliberately small, preventive, and atomic so it can be enforced automatically; it is the floor, not the target. Higher-risk pipelines layer on the enhanced controls in Sections 6–9.

### 2.1 Risk tiers
Every pipeline is assigned a tier that determines which enhanced controls apply on top of the baseline. When in doubt the higher tier applies, and **any pipeline whose agent can reach production or real secrets is Tier 1** regardless of what it deploys.

| Tier | Applies to | What is required |
|---|---|---|
| **Tier 1 — Critical / regulated** | Production with sensitive or regulated data, internet-facing services, and ANY pipeline with an agent that can reach production or real secrets. | Full baseline plus all enhanced controls (Sec. 6–9). Human gates and the Rule of Two are mandatory. |
| **Tier 2 — Standard production** | Internal production without regulated or sensitive data. | Full baseline plus enhanced controls per risk; selected SHOULD controls may be risk-accepted by exception. |
| **Tier 3 — Non-production / sandbox** | Development, test, and sandbox pipelines with no production deployment, no real secrets, and no real tenant data. | Baseline hygiene only (B1, B4, B7, B8, B11, B13); controls predicated on production, secrets, or tenant data are relaxed. |

### 2.2 Baseline control set
Each row is the production floor. The full control (with its acceptance Check and framework mapping) lives where the **Ref** column points; this table does not restate it.

| ID | Baseline requirement | Ref |
|---|---|---|
| **B1** | Protected branches require review; no direct push to protected branches | HRD-2, §8.1 |
| **B2** | No autonomous agent merge or deploy to production (human gate required) | AGT-4.1 |
| **B3** | Agents obey the Rule of Two (never untrusted input + secrets + external action together) | AGT-5.1 |
| **B4** | No hardcoded or shared secrets; resolved from a vault | SCR-1, SCR-2 |
| **B5** | Short-lived, federated workload identity; no static long-lived keys | IAM-2 |
| **B6** | Ephemeral, single-tenant runners destroyed after each job | HRD-1 |
| **B7** | SAST, SCA, and secret scanning enforced as blocking gates | §8.4 |
| **B8** | Third-party actions and plugins pinned to an immutable digest / commit SHA | SUP-4 |
| **B9** | Artifacts signed and verified before deploy | SUP-3 |
| **B10** | SBOM generated and retained per release | SUP-2 |
| **B11** | Tenant-tagged, immutable audit logging to a SIEM | MTN-3 |
| **B12** | Least-privilege pipeline tokens; no standing production write from PR context | IAM-3 |
| **B13** | Pipeline definitions reviewed as code | HRD-2 |
| **B14** | Documented, controlled break-glass path | BG-1 (§2.3) |

### 2.3 Break-glass and emergency change
A mandatory human gate with no sanctioned emergency path guarantees shadow pipelines during incidents. Break-glass is the sanctioned, audited alternative.

**BG-1 — Provide a controlled break-glass deployment path.** An emergency path MUST exist that bypasses non-essential gates only under defined conditions. It MUST require explicit, attributable authorization, be restricted to break-glass-entitled identities, raise a high-severity alert on use, and be reconciled by a mandatory post-incident review. It MUST NOT bypass artifact signing (SUP-3), secret hygiene (SCR-1), or audit logging (MTN-3).
- *Check:* A break-glass runbook exists; each invocation produces an alert and a completed post-incident review.
- *Mapping:* CICD-SEC-1; NIST AC-6, CM-3, IR-4.

### 2.4 How the baseline is enforced
The baseline is enforced preventively as policy-as-code, not by self-attestation (Section 13). **Exceptions to B2, B3, and B4 require explicit CISO approval and a documented compensating control and MUST NOT be granted under delegated authority.**

---

## 3. CI/CD Security Risks and Why They Matter

A pipeline is attractive to adversaries because it is a trusted, automated, high-privilege channel into production, and one compromise can propagate to every downstream consumer. SolarWinds (a poisoned build system distributing malware to thousands) and Codecov (secrets harvested from environment variables across thousands of pipelines) are the archetypes. The OWASP Top 10 CI/CD Security Risks below remain fully in force in AI-enabled pipelines and are the baseline this standard builds on.

| Risk | Why it matters | Typical blast radius |
|---|---|---|
| **CICD-SEC-1** Insufficient flow control | Code or artifacts reach production without review. | Unreviewed change in production |
| **CICD-SEC-2** Inadequate IAM | Excessive or unmanaged identities give broad reach. | Lateral movement across the chain |
| **CICD-SEC-3** Dependency-chain abuse | Malicious or confused dependencies execute in the build. | Code execution on build/dev machines |
| **CICD-SEC-4** Poisoned pipeline execution | Unreviewed CI config runs attacker commands. | Job secrets and build environment |
| **CICD-SEC-5** Insufficient PBAC | Pipelines over-permissioned beyond any step's need. | Credential theft, artifact poisoning |
| **CICD-SEC-6** Insufficient credential hygiene | Hardcoded, shared, exposed, or unrotated secrets. | Direct access to high-value systems |
| **CICD-SEC-7** Insecure system configuration | Unhardened or unpatched CI/CD systems. | System compromise; OS access |
| **CICD-SEC-8** Ungoverned 3rd-party use | High-privilege integrations with little oversight. | Expanded surface; code exfiltration |
| **CICD-SEC-9** Improper artifact integrity validation | Tampered artifacts flow to production undetected. | Supply-chain compromise of releases |
| **CICD-SEC-10** Insufficient logging & visibility | Attacker activity proceeds undetected. | Undetected breach; failed investigation |

**The new risk class.** Embedding an autonomous agent opens a second surface that overlaps with, but is distinct from, the list above: the agent ingests untrusted context, interprets it as language, and decides which tools to invoke, often while holding the job's secrets and tool access. Prompt injection, credential harvesting, non-deterministic generated code, autonomous merge, and excessive agency are treated in full in [Section 7](#7-a-new-class-of-risk-agentic-ai-in-cicd).

---

## 4. Components and Their Controls

Every system below is in scope and MUST appear in the pipeline inventory (MTN-1). This single table replaces the separate component and summary tables of earlier drafts; framework mappings are in Appendix A.

| Component | Principal risk | Governing controls | Implementation & isolation note |
|---|---|---|---|
| Source control (SCM) | Unreviewed change to production | B1, HRD-2 | Entra-federated SCM; per-tenant repo boundaries and branch protection |
| CI build servers & runners | Poisoned pipeline execution | B6, HRD-1 | Ephemeral single-tenant runners; managed identity; no shared state |
| Artifact & container registries | Tampered / substituted artifact | B9, SUP-3 | ACR content trust / cosign; tenant-scoped namespaces |
| Dependencies | Supply-chain abuse | B8, B10, SUP-1/4 | Private feeds; digest pinning; SBOM |
| Secrets stores / key vaults | Credential theft | B4, SCR-1/2/3 | Key Vault; per-tenant scoped paths; workload identity |
| Deployment targets | Unauthorized / autonomous deploy | B2, AGT-4.1 | Environment protection rules; per-tenant network/identity segmentation |
| Model endpoints / AI gateway | Output injection; cost abuse | AGT-6.1, CON-1 | AI gateway with output policy and per-tenant budgets |
| Agent runtime environment | Prompt injection; credential harvest | B3, AGT-1/2/5 | Sandboxed, secret-free, single-tenant ephemeral runner |
| Training / fine-tuning data | Data and model poisoning | DAT-1 | Versioned, provenance-tracked datasets; integrity checks |
| Tenant configuration store | Cross-tenant breach | MTN-2 | Segmented, tamper-evident; authoritative isolation boundary |
| Observability / log pipeline | Undetected compromise | B11, MTN-3, SCR-4 | Sentinel; tenant-tagged immutable logs; no secrets/PII in logs |

---

## 5. Multimodel and Multitenant Challenges

The combination of multiple models, shared infrastructure, and autonomous agents creates challenges none of the three produce alone. Each maps to a control response elsewhere in this standard.

| Challenge | Why it is new or amplified | Control response |
|---|---|---|
| Shared runner contamination | Residue from one job (credentials, files, poisoned binaries) influences the next, across tenants. | Ephemeral single-tenant runners (HRD-1, B6) |
| Cross-tenant secret leakage | Broadly scoped tokens or shared logs let one harvested secret unlock many tenants. | Tenant-scoped, short-lived secrets; no shared logs (SCR-1/2, MTN-3) |
| Model output injection | One model's output becomes another step's input and is wrongly trusted. | Validate model output before use (AGT-6.1) |
| Non-deterministic agent behavior | The same input yields different tool calls, breaking the determinism traditional controls assume. | Constrain capability, not behavior (AGT-5.1, human gates) |
| Audit and traceability gaps | Hard to reconstruct which identity/model/prompt/tenant did what. | Per-agent identity; per-tenant immutable logs (AGT-5.2, MTN-3) |

### 5.1 The multitenancy trust model
Isolation controls are meaningless without a stated assumption. **This standard assumes tenants are mutually untrusted and adopts an assume-breach posture:** a compromise in one tenant's pipeline MUST NOT reach another's code, secrets, artifacts, or runtime. Required strength follows the tier — Tier 2 MAY use logical isolation (namespace, scoped identity, network segmentation) where those controls are enforced and audited; Tier 1 exposed to hostile-tenant risk MUST use hard isolation (dedicated identity planes and, where warranted, dedicated infrastructure). This is the rationale behind the MTN controls in Section 9.

---

## 6. Types of CI/CD Pipelines Covered

Controls apply to all pipeline types below; type-specific controls are stated explicitly. The distinction matters because trust model and blast radius differ — an agentic pipeline that can merge its own code carries risks a test pipeline does not.

| Pipeline type | What it does | Distinctive risk emphasis |
|---|---|---|
| Traditional application | Builds, tests, deploys conventional software. | Flow control, supply chain, credential hygiene (CICD-SEC-1/3/6). |
| ML training | Ingests data; trains or fine-tunes models. | Data and model poisoning, dataset provenance (LLM04; DAT-1). |
| Model deployment | Packages, signs, serves models to endpoints/gateway. | Artifact integrity, model supply chain (LLM03; CICD-SEC-9). |
| Agentic execution | An AI agent triggers builds, opens/merges PRs, or deploys. | Prompt injection, excessive agency, autonomous merge (LLM01/06). |

**Composite pipelines** inherit the controls of every type they contain and take the highest tier any component triggers; the strictest applicable control governs.

---

## 7. A New Class of Risk: Agentic AI in CI/CD

> **Agentic AI as a first-class threat.**

An agentic pipeline is one in which an AI agent — for example an AI coding action inside a CI runner — ingests repository context, interprets natural-language input, and decides which tools to invoke, often holding the secrets and tool access of the job it runs in. The pattern is consistent across vendors: events supply context, some context is untrusted user input, that content is embedded in a prompt, the output is treated as actionable, and the agent runs with access to secrets and tools. When the boundary between untrusted content and control instruction fails, the workflow becomes an attacker-steerable agent inside the repository.

### 7.1 Reference incident: the agent that read its own secrets

> **Case study — AI coding agent as a GitHub Action (disclosed June 2026).**
> Microsoft Threat Intelligence found an AI coding agent running as a GitHub Action could expose workflow secrets when processing untrusted GitHub content. Its Bash path ran in a namespace sandbox with a scrubbed environment, but its file-read tool did not share that isolation — it made direct, in-process calls with full access to the runner's environment variables.
>
> A payload hidden in an HTML comment (invisible when rendered, readable in raw markdown) steered the agent to read `/proc/self/environ`. Framed as a "compliance review" and told to cut the first characters before printing, it laundered the value past the model's refusal layer and the platform's pattern-based secret scanner, printing the unscrubbed API key.
>
> The vendor mitigated by unconditionally blocking the file-read tool from sensitive `/proc` files. The durable lesson — and the basis for the controls below — is broader: **an AI workflow that processes untrusted input must not also hold secret access and an external action channel at the same time.**

### 7.2 The agentic risk taxonomy and required controls

#### 7.2.1 Prompt injection from pipeline inputs
**Threat.** Any text the agent reads — issue/PR bodies, comments, commit messages, file contents, even HTML-comment text — can carry instructions overriding its task. This is the agentic form of poisoned pipeline execution: the attacker needs no write access, only to get text in front of the model.

**AGT-1.1 — Treat all pipeline-readable content as untrusted data, never instruction.** The agent system prompt MUST name every readable surface and state that all such content is untrusted author input, not instruction — even when phrased as one, quoted, or wrapped in markdown.
- *Check:* System prompt names readable surfaces and labels them untrusted; verified by injection test.
- *Mapping:* LLM01; CICD-SEC-4; NIST SI-10; SSDF PW.1.

**AGT-1.2 — Pin the agent task and refuse out-of-scope actions.** The system prompt MUST state the single job the workflow performs and instruct refusal of anything outside that scope. Hardened prompts are defense in depth, not the sole control.
- *Check:* An out-of-scope instruction in a test issue is refused.
- *Mapping:* LLM01 / LLM06 / LLM07; NIST AC-6; SSDF PW.1.

#### 7.2.2 Agent credential abuse and harvesting
**Threat.** An agent with file-read or shell tools and runner-environment access can read secrets and exfiltrate them; models may refuse to print an obvious credential, but attackers launder the value to defeat both the refusal layer and pattern-based scanners.

**AGT-2.1 — Deny agent tools access to credential-bearing paths.** Agent file-read and execution tools MUST be denied access to process/environment files (e.g. `/proc/self/environ`) and secret mounts, and MUST share one isolation boundary and scrubbed environment; an isolation gap in any single tool voids the control.
- *Check:* An attempt to read an environment/secret path via any agent tool is blocked and alerts.
- *Mapping:* LLM02 / LLM06; CICD-SEC-6; NIST SC-39, AC-6; Azure: Key Vault references, managed identity.

**AGT-2.2 — Keep long-lived secrets out of the agent runtime.** Secrets the agent does not strictly require MUST NOT be present in the runner environment while untrusted input is processed; required secrets MUST be short-lived, brokered just-in-time, and scoped to the single operation.
- *Check:* Runner environment inspected during untrusted-input processing holds no long-lived secrets.
- *Mapping:* CICD-SEC-6; NIST IA-5; Azure: workload identity federation, Key Vault.

#### 7.2.3 Non-deterministic generated code
**Threat.** Model-authored code is non-deterministic and may introduce subtle defects or attacker-directed changes (e.g. an invisible script appended to a doc file). Trusting generated output lets it flow into artifacts unreviewed.

**AGT-3.1 — Subject agent-generated changes to the same gates as human changes.** Agent-produced code, configuration, or artifacts MUST pass the same review, SAST/SCA, secret-scanning, and integrity controls as human-authored changes, and MUST be attributed to the agent in version history.
- *Check:* Agent-authored PRs traverse identical required checks; agent attribution visible in history.
- *Mapping:* LLM05; CICD-SEC-1 / CICD-SEC-9; SSDF PW.7, PW.8; NIST SA-11.

#### 7.2.4 Autonomous merge and deployment
**Threat.** An agent that can merge its own PRs or deploy directly removes the external verification flow control depends on, letting a single manipulable actor ship to production.

**AGT-4.1 — Require a human gate before agent changes reach a protected branch or production.** An agent MUST NOT merge to a protected branch or deploy to production without approval by a human (or an independent non-agent control) that did not author the change; auto-merge rules MUST exclude agent-authored changes.
- *Check:* No production deploy or protected-branch merge is attributable solely to an agent identity.
- *Mapping:* LLM06; CICD-SEC-1; NIST CM-3, AC-6; Zero Trust.

#### 7.2.5 Excessive agency and the Rule of Two
**Threat.** The root enabler of the reference incident was an agent that simultaneously processed untrusted input, held secret access, and could act externally. Any agent holding all three becomes a remote-code path the moment an injection succeeds. The rule below originates as Meta's "Agents Rule of Two."

**AGT-5.1 — Apply the Agents Rule of Two.** No agentic workflow MAY hold more than two of: (a) processing untrusted input; (b) access to sensitive systems or secrets; (c) the ability to change state or communicate externally. A workflow needing all three MUST be decomposed into separately governed stages with a trust boundary between them. This is baseline control **B3**; exceptions require explicit CISO approval and a compensating control (Section 13).
- *Check:* Each agentic workflow's recorded capability set holds at most two of the three.
- *Mapping:* LLM06; CICD-SEC-5; NIST AC-6; Zero Trust: assume breach.

**AGT-5.2 — Give every agent a distinct, scoped, revocable identity.** Each agent MUST act under its own identity (not a shared service account or static key), scoped to the tools and resources it needs, individually attributable in logs, and independently revocable; tool access MUST be governed by per-tool authorization.
- *Check:* Each agent maps to one IdP identity with scoped tool permissions; revoking it disables only that agent.
- *Mapping:* LLM06; CICD-SEC-2; NIST IA-2, IA-9, AC-6; Azure: Entra ID workload identities.

#### 7.2.6 Agent-to-agent and model supply-chain trust
**Threat.** In multimodel pipelines, agents and models consume one another's output and depend on third-party models, SDKs, and tool servers; a compromised upstream propagates downstream, and one agent's output can be a vector into another.

**AGT-6.1 — Validate inter-agent and model output at every trust boundary.** Output from one model or agent that becomes input to another MUST be validated against an explicit schema or policy before use, and MUST NOT be executed or forwarded as instruction without that validation.
- *Check:* Inter-model handoffs pass a schema/policy validation step before downstream use.
- *Mapping:* LLM05; CICD-SEC-9; NIST SI-10.

**AGT-6.2 — Govern the model and tool supply chain.** Models, agent SDKs, and connected tool servers MUST be sourced from vetted origins, version-pinned, and inventoried; integrity SHOULD be verified by signature where supported.
- *Check:* A model/SDK/tool-server inventory exists with pinned versions and vetted sources.
- *Mapping:* LLM03; CICD-SEC-3 / CICD-SEC-8; NIST SR-3, SR-11; SLSA.

### 7.3 Adversary techniques (MITRE ATLAS)
The reference incident maps to MITRE ATLAS (as cited by the disclosing researchers), which teams SHOULD use when threat-modeling agentic pipelines.

| ATLAS tactic | Technique | Control |
|---|---|---|
| Resource development | AML.T0065 LLM prompt crafting | AGT-1.1, AGT-1.2 |
| Execution | AML.T0051 LLM prompt injection | AGT-1.1, AGT-5.1 |
| Execution | AML.T0053 AI agent tool invocation | AGT-2.1, AGT-5.2 |
| Defense evasion | AML.T0054 LLM jailbreak | AGT-1.2, AGT-2.1 |
| Credential access | AML.T0098 AI agent tool credential harvesting | AGT-2.1, AGT-2.2 |
| Exfiltration | AML.T0057 LLM data leakage | AGT-5.1, AGT-2.2 |

---

## 8. Security Controls by Pipeline Stage

The pipeline lifecycle and the control domains (Section 9) are two views of one control set. Rather than restate the controls, this table maps each stage to its principal threats and the controls that apply; framework mappings are in Appendix A.

| Stage | Principal threats | Controls that apply |
|---|---|---|
| **8.1** Commit & source control | Direct-to-prod push, branch-protection bypass, secrets in history, injection text planted for agents | B1, B13, HRD-2, SCR-1 |
| **8.2** Build & compilation | Poisoned pipeline execution, build-node compromise, secret theft from job env | B6, HRD-1, IAM-3 |
| **8.3** Dependency & supply chain | Dependency confusion, typosquatting, hijacked or malicious transitive packages | B8, B10, SUP-1, SUP-2, SUP-4 |
| **8.4** Static analysis (SAST/SCA/secrets) | Vulnerable code, leaked secrets, known-vulnerable dependencies reaching later stages | B7, AGT-3.1 |
| **8.5** Containerization & image build | Untrusted base images, secrets in layers, image tampering | SUP-1, SUP-3, HRD-3 |
| **8.6** Artifact storage & signing | Tampered/substituted artifacts, unsigned artifacts, illegitimate uploads | B9, SUP-3 |
| **8.7** Dynamic testing (DAST/IAST) | Runtime-only vulnerabilities; for AI surfaces, prompt-injection and output-handling flaws | DAST + LLM red-team gate; AGT-1.x |
| **8.8** Deployment (staging & prod) | Unauthorized, unreviewed, or autonomous deployment of unverified artifacts | B2, AGT-4.1, MTN-2 |
| **8.9** Runtime & post-deployment | Undetected compromise, configuration drift, anomalous agent or pipeline behavior | B11, MTN-3, CON-2 |

*Stages are sequential but controls are cumulative: a weakness at any stage undermines those that follow.*

---

## 9. Security Controls by Domain

These are the normative controls, defined once here (and, for agentic controls, in Section 7). Sections 2 and 8 reference them. IDs are domain-scoped and stable; each control carries an acceptance **Check**.

### 9.1 Identity and access (IAM)

**IAM-1 — Federate all pipeline identities to one IdP with MFA.** Human and machine identities across SCM, CI, registries, and clouds MUST be federated to one IdP with MFA; local and shared accounts MUST be eliminated.
- *Check:* Identity inventory shows no local/shared accounts; all access via the federated IdP with MFA.
- *Mapping:* CICD-SEC-2; NIST IA-2, AC-2; Azure: Entra ID.

**IAM-2 — Use short-lived, workload-scoped credentials.** Pipelines MUST authenticate with short-lived federated workload identities, not static keys; tokens MUST be scoped to the minimum permission and lifetime the step needs.
- *Check:* No static long-lived keys in pipeline config; tokens expire within the defined maximum.
- *Mapping:* CICD-SEC-6; NIST IA-5, AC-6; Azure: workload identity federation.

**IAM-3 — Enforce least privilege per pipeline and per step.** Each pipeline/step MUST hold only the permissions it needs; standing access to production and secrets MUST be minimized; PR-triggered contexts MUST NOT hold production write.
- *Check:* Permission review shows no step exceeding need; no production write from PR context.
- *Mapping:* CICD-SEC-5; NIST AC-6.

**IAM-4 — Manage the agent identity lifecycle.** Agent identities MUST be issued through a governed process, rotated on a defined cycle, and deprovisioned on retirement; stale/orphaned agent identities MUST be detected and removed.
- *Check:* Every active agent identity maps to a current owner and agent; no orphans in the periodic review.
- *Mapping:* CICD-SEC-2; NIST IA-4, IA-5, AC-2; Azure: Entra ID lifecycle.

### 9.2 Secrets management (SCR)

**SCR-1 — Prohibit hardcoded and shared secrets.** Secrets MUST NOT be hardcoded in code, pipeline definitions, or images, nor shared across environments, tenants, or workflows.
- *Check:* Secret scanning passes; no secret value reused across environments or tenants.
- *Mapping:* CICD-SEC-6; NIST IA-5; Azure: Key Vault.

**SCR-2 — Broker secrets from a vault, just in time.** Secrets MUST be retrieved at runtime from a managed vault via workload identity, injected only for the step that needs them, and never written to shared logs.
- *Check:* Secrets resolve from the vault at runtime; log inspection shows no secret material.
- *Mapping:* NIST IA-5, SC-12, SC-28; Azure: Key Vault references.

**SCR-3 — Rotate and revoke on cycle and on exposure.** Secrets MUST be rotated on a defined cycle and immediately on suspected exposure; provider-side usage MUST be monitored with alerting on anomalous use.
- *Check:* Rotation records meet cadence; anomalous-use alerts are configured at the provider.
- *Mapping:* CICD-SEC-6; NIST IA-5, SI-4, AU-6.

**SCR-4 — Keep secrets and sensitive prompt/response content out of logs.** Logs and telemetry MUST NOT capture secret material or sensitive prompt/response content (which may contain credentials or personal data); such fields MUST be redacted or excluded at source.
- *Check:* Sampled logs contain no secrets or sensitive prompt/response payloads; redaction is enforced at source.
- *Mapping:* LLM02; CICD-SEC-6; NIST AU-9, SI-12.

### 9.3 Pipeline hardening and recoverability (HRD)

**HRD-1 — Use ephemeral, isolated runners.** Build and agent runners MUST be ephemeral and single-tenant, with no shared mutable state, and destroyed after each run.
- *Check:* Runner telemetry shows per-job creation and destruction; no persistent shared runner in scope.
- *Mapping:* CICD-SEC-4, CICD-SEC-7; NIST CM-2, SC-39.

**HRD-2 — Treat pipeline definitions as protected code.** Pipeline/workflow definitions MUST be version-controlled, reviewed, and protected like application code; changes MUST require review.
- *Check:* Workflow files live in protected paths; changes require review before merge.
- *Mapping:* CICD-SEC-4; NIST CM-3; SSDF PO.3.

**HRD-3 — Harden and patch CI/CD systems against a baseline.** All CI/CD systems MUST be configured to a hardening baseline, patched on a defined cadence with severity-based SLAs, and assessed for drift; default credentials MUST be disabled.
- *Check:* Posture assessment conforms to baseline; no overdue critical patches; defaults disabled.
- *Mapping:* CICD-SEC-7; NIST CM-6, RA-5; Azure: Defender for Cloud posture.

**HRD-4 — Make the pipeline platform recoverable.** The platform MUST be treated as production infrastructure: configuration stored as code and backed up, defined RTO/RPO, periodically tested recovery, and re-issuable agent/workload identities after a recovery event.
- *Check:* A tested recovery procedure with stated RTO/RPO exists; last test within the defined interval.
- *Mapping:* NIST CP-9, CP-10, CM-2; Azure: IaC, backup.

### 9.4 Supply-chain security (SUP)

**SUP-1 — Pin dependencies and resolve from trusted feeds.** Dependencies MUST be pinned to verified versions/hashes and resolved from trusted, prioritized internal feeds; known-vulnerable components MUST be blocked at the gate.
- *Check:* Dependencies are hash-pinned; the gate blocks a known-vulnerable test dependency.
- *Mapping:* CICD-SEC-3; NIST SR-3, SR-4; SSDF PW.4.

**SUP-2 — Generate, retain, and consume an SBOM.** Every release MUST produce a retained SBOM, used to assess exposure when new vulnerabilities are disclosed.
- *Check:* Each release has a stored SBOM; SBOMs are queried during vulnerability triage.
- *Mapping:* SSDF PS.3, RV.1; NIST SR-4; CICD-SEC-9.

**SUP-3 — Sign artifacts and verify before deployment.** Artifacts MUST be signed at build and verified before deployment; unsigned or unverifiable artifacts MUST be rejected.
- *Check:* The deploy gate rejects an unsigned test artifact; signatures are verified in the pipeline.
- *Mapping:* CICD-SEC-9; NIST SI-7, SR-4; SSDF PS.2; SLSA provenance.

**SUP-4 — Pin third-party actions and plugins to an immutable digest.** Third-party actions, plugins, and reusable workflows MUST be pinned to an immutable commit SHA or digest; floating tags MUST NOT be used for external steps.
- *Check:* Pipeline scan finds no floating external references; all third-party steps pinned to a digest/SHA.
- *Mapping:* CICD-SEC-8; NIST SR-3, SR-11; SLSA.

**SUP-5 — Meet a build-provenance level for production.** Production build provenance MUST meet SLSA Build Level 3 (non-falsifiable provenance generated by a hardened build service).
- *Check:* Production builds emit verifiable SLSA Build L3 provenance.
- *Mapping:* CICD-SEC-9; NIST SR-4; SLSA Build L3.

### 9.5 Data and model integrity (DAT)

**DAT-1 — Establish dataset and model provenance and integrity.** Training and fine-tuning datasets and model artifacts MUST be versioned, provenance-tracked, and integrity-verified before use; data from untrusted or unverified sources MUST NOT enter training without review, to mitigate data and model poisoning.
- *Check:* Datasets and models carry version and provenance records; integrity is verified before a training/serving run.
- *Mapping:* LLM04; CICD-SEC-3; NIST SR-4, SI-7; SSDF PW.4.

### 9.6 Consumption and abuse controls (CON)

**CON-1 — Enforce per-tenant budgets and rate limits at the AI gateway.** Model invocation MUST be bounded by per-tenant token/compute budgets and request rate limits at the AI gateway, containing cost-based denial of service and runaway spend.
- *Check:* The gateway enforces per-tenant budgets/limits; exceeding them is throttled and alerted.
- *Mapping:* LLM10; NIST SC-5; Azure: AI gateway policy.

**CON-2 — Bound agent iteration depth and tool-call volume.** Agentic runs MUST bound recursion/iteration depth and total tool calls per run, and alert on anomalous consumption.
- *Check:* A run exceeding the configured depth/tool-call limit is halted and alerts.
- *Mapping:* LLM10; NIST SI-4, SC-5.

### 9.7 Multitenancy isolation (MTN)

These implement the trust model in Section 5.1: tenants are mutually untrusted and isolation strength follows the tier.

**MTN-1 — Maintain a tenant-aware pipeline inventory.** A current inventory of all pipeline systems, their owners, and tenant assignments MUST be maintained as the basis for isolation, accountability, and visibility.
- *Check:* Inventory exists, names an owner and tenant per component, and is current within the review interval.
- *Mapping:* CICD-SEC-10; NIST CM-8.

**MTN-2 — Enforce isolation by namespace, identity, and network, scaled to tier.** Tenants MUST be separated by repository/registry namespace, scoped identity and secret paths, and network segmentation. Tier 1 pipelines exposed to hostile-tenant risk MUST additionally use hard isolation (dedicated identity planes and, where warranted, dedicated infrastructure). No component may cross tenant boundaries except through a governed control plane.
- *Check:* Isolation test confirms no cross-tenant reach; Tier 1 hostile-risk pipelines use dedicated identity planes.
- *Mapping:* NIST SC-7, SC-2, AC-4; Zero Trust: assume breach.

**MTN-3 — Tag and segregate per-tenant audit logs.** Security-relevant events MUST be tagged with the acting tenant and identity and stored in immutable, tenant-segregated logs sufficient to reconstruct any action.
- *Check:* Sampled events carry tenant+identity tags; logs are immutable and segregated per tenant.
- *Mapping:* CICD-SEC-10; NIST AU-2, AU-9, AU-12.

---

## 10. Secure Agentic CI/CD Pipeline: Reference Architecture

This reference shows how an AI coding agent participates safely. It enforces the core principle from Section 7 — the moment an agent processes untrusted input it must not also hold both secret access and an external action channel — by decomposing the agent's work into stages separated by trust boundaries, with a human gate before any protected change.

### 10.1 Trust boundaries
Three boundaries divide the pipeline: untrusted external content vs the agent's instruction context; the agent's reasoning environment vs credential-bearing resources; and proposed changes vs protected branches and production. No single component spans all three.

### 10.2 Control flow

| Stage | What happens | Controls | Trust posture |
|---|---|---|---|
| 1. Ingest | Event delivers context; untrusted fields labeled. | AGT-1.1; SI-10 | All external content = untrusted |
| 2. Reason (sandboxed) | Agent reasons in an isolated, secret-free runtime. | AGT-2.2, AGT-5.1; SC-39 | No standing secrets; no external write |
| 3. Propose | Agent emits a PR; no merge authority. | AGT-4.1; CM-3 | Output validated; agent-attributed |
| 4. Verify | SAST/SCA, secret scan, signature, policy gates run. | AGT-3.1; SA-11 | Proposal treated like any change |
| 5. Human gate | Independent human/non-agent control approves. | AGT-4.1; AC-6 | External verification required |
| 6. Promote | Verified, signed artifact deploys via scoped identity. | IAM-2, SUP-3 | Just-in-time, least-privilege |
| 7. Observe | Per-agent, per-tenant immutable telemetry to SIEM. | MTN-3, SCR-4; AU-12 | Fully attributable; no secrets logged |

### 10.3 Monitoring and rollback
- **Monitoring.** Agent tool invocations, secret-access attempts, injection signatures (instruction-like text in untrusted fields), and anomalous external calls are logged per agent identity and alerted on.
- **Rollback triggers.** A successful injection, an agent attempt to read a denied path, an unsigned artifact, or an attempted autonomous merge MUST halt the pipeline, revoke the agent's active credentials, and roll back to the last verified state.

### 10.4 Platform recovery
The pipeline is itself production infrastructure and is recoverable per **HRD-4**: configuration stored as code and backed up, RTO/RPO defined and tested, and agent/workload identities re-issuable after recovery without weakening the controls above.

---

## 11. Responsibilities (RACI)

R = Responsible, A = Accountable, C = Consulted, I = Informed. Where A/R is shown for two roles, the accountable role holds final accountability and the second executes.

| Activity | Platform Eng | DevSecOps | AI/ML Eng | Tenant Op | CISO/Gov |
|---|---|---|---|---|---|
| Platform hardening & recovery | A/R | C | I | I | I |
| Scanning & policy gates | C | A/R | C | I | I |
| Identity & secrets management | R | C | I | C | A |
| Agentic AI controls (Sec. 7) | C | R | A/R | I | C |
| Data & model integrity | C | C | A/R | I | I |
| Tenant isolation config | C | C | I | A/R | C |
| Baseline enforcement (policy-as-code) | R | A/R | C | I | C |
| Break-glass authorization | C | C | I | C | A/R |
| Exception approval | I | C | C | I | A/R |
| Metrics (Sec. 12) | C | R | C | I | A |
| Incident response | R | R | C | C | A |

---

## 12. Measuring Effectiveness

Each metric names a data source, an accountable owner, an SLO target, and the action on breach. Metrics are reported to the CISO governance function on the review cadence. Tenants may set stricter targets, not weaker ones.

| Metric | Source | Owner | SLO target | If breached |
|---|---|---|---|---|
| MTTD for secret exposure | SIEM / scanner | DevSecOps | < 24h, trending down | Incident review; tighten detection |
| Pipelines with enforced SAST/SCA gates | Policy-as-code | DevSecOps | 100% in-scope | Block deploys until gate added |
| Agent merges to protected branches | SCM audit | AI/ML Eng | 0 autonomous | Sev-1; revoke agent merge rights |
| Cross-tenant access incidents | SIEM | Platform Eng | 0 | Sev-1 incident response |
| SBOM coverage | Release records | DevSecOps | 100% of releases | Hold release; remediate |
| Secret rotation compliance | Vault | Platform Eng | ≥ 98%; 0 overdue high-value | Force rotation; exception review |
| Ephemeral runner compliance | Runner telemetry | Platform Eng | 100% in-scope jobs | Quarantine persistent runners |
| Rule-of-Two compliance | Workflow inventory | AI/ML Eng | 100%; exceptions time-bound | Decompose workflow; log exception |
| Third-party steps digest-pinned | Pipeline scan | DevSecOps | 100% external steps | Fail pipeline until pinned |
| Break-glass invocations reconciled | Change log | CISO/Gov | 100% post-reviewed | Escalate unreconciled use |

---

## 13. Governance, Enforcement, and Exceptions

A standard that relies on goodwill fails. This section defines technical enforcement, exception handling, and assurance.

### 13.1 Enforcement model
Enforcement is preventive first: controls are enforced as policy-as-code wherever the platform allows, so a non-conforming pipeline cannot deploy rather than merely being flagged afterward. Detective controls supplement but do not replace preventive enforcement for baseline controls.

**GOV-1 — Enforce the baseline as policy-as-code.** Baseline controls (Section 2) MUST be enforced through preventive mechanisms — required status checks, branch protection, admission control, and pipeline policy engines — such that a pipeline failing a baseline control cannot deploy to production. Detective-only enforcement is insufficient for baseline controls.
- *Check:* A deliberately non-conforming test pipeline is blocked from production by automated policy, not manual review.
- *Mapping:* CICD-SEC-1/7; NIST CM-2, CM-3, SI-4; Azure: Defender for Cloud (DevOps security), admission control.

### 13.2 Exception process
**GOV-2 — Govern deviations through a bounded exception process.** Any deviation from a MUST control requires a recorded exception naming a compensating control, approved by the accountable owner, expiring within 90 days, and held in an exception register reviewed at expiry. Exceptions to baseline controls B2, B3, and B4 require explicit CISO approval and a documented compensating control and MUST NOT be granted under delegated authority.
- *Check:* Every active deviation has a register entry with compensating control, approver, and expiry; B2/B3/B4 entries carry CISO approval.
- *Mapping:* NIST RA-7, PM-4, CA-5; Zero Trust: verify explicitly.

### 13.3 Assurance cadence
Conformance is assured continuously (the policy-as-code enforcement in 13.1 and the metrics in Section 12) and periodically (a scheduled audit against this standard, the exception-register review, and the annual review in the document-control table). Findings feed the next revision.

---

## Appendix A: Framework Control Mapping

Crosswalk from this standard's control domains to the normative frameworks. Mappings are indicative, not exhaustive; "—" means the framework does not address that domain directly.

| This standard | NIST 800-53 Rev 5 | SSDF | OWASP CI/CD | OWASP LLM | Azure |
|---|---|---|---|---|---|
| Baseline (Sec. 2) | CM-3, AC-6, IA-5 | PO.3 | SEC-1/4/6 | LLM01/06 | Defender for Cloud (DevOps security) |
| Identity & access (IAM) | IA-2/4/5, AC-2/6 | PO.3 | SEC-2, SEC-5 | LLM06 | Entra ID, workload identity |
| Secrets (SCR) | IA-5, SC-12/28, AU-9, SI-12 | PS.1 | SEC-6 | LLM02 | Key Vault |
| Hardening & recovery (HRD) | CM-2/6, SC-39, CP-9/10 | PO.3, PO.5 | SEC-4, SEC-7 | — | Defender for Cloud, IaC, backup |
| Supply chain (SUP) | SR-3/4/11, SI-7 | PS.2/3, PW.4, RV.1 | SEC-3, SEC-8, SEC-9 | LLM03 | ACR, SBOM, digest pinning |
| Data & model integrity (DAT) | SR-4, SI-7 | PW.4 | SEC-3 | LLM04 | Dataset versioning/provenance |
| Consumption (CON) | SC-5, SI-4 | — | — | LLM10 | AI gateway budgets/limits |
| Agentic AI (AGT, Sec. 7) | SI-10, AC-6, IA-9, SC-39 | PW.1 | SEC-1/4/5 | LLM01/05/06/07 | Defender for Cloud (AI) |
| Multitenancy (MTN) | SC-7, SC-2, AC-4, AU-9 | — | SEC-10 | — | Network/identity segmentation |
| Logging & visibility (Sec. 8.9) | AU-2/6/12, SI-4 | RV.1 | SEC-10 | — | Microsoft Sentinel |
| Governance (GOV, Sec. 13) | RA-7, PM-4, CA-5, CM-3 | PO.1 | SEC-1 | — | Policy-as-code, admission control |

> **Rev 5 note:** the former SA-12 (Supply Chain Protection) was withdrawn in NIST SP 800-53 Rev 5 and incorporated into the SR (Supply Chain Risk Management) family; supply-chain controls here cite SR-3, SR-4, and SR-11.

---

## Appendix B: Glossary

| Term | Definition |
|---|---|
| Agentic AI | An AI system that interprets natural-language context and autonomously decides which tools or actions to invoke. |
| Agents Rule of Two | Meta's design rule that no AI workflow should hold all three of: untrusted input, secret/system access, and external action. |
| Baseline (minimum production) | The non-negotiable controls that gate every production deployment regardless of tier (Section 2). |
| Break-glass | A controlled, audited emergency path that bypasses non-essential gates under defined authorization, with mandatory post-incident review. |
| CI/CD | Continuous integration and continuous delivery/deployment: the automated path from commit to production. |
| Data/model poisoning | Manipulation of training data or models to corrupt behavior (OWASP LLM04). |
| Digest pinning | Referencing a third-party action, image, or dependency by an immutable commit SHA or content digest rather than a movable tag. |
| Ephemeral runner | A build/agent environment created for one job and destroyed afterward, leaving no reusable state. |
| Excessive agency | An agent holding more autonomy, tools, or permissions than its task requires (OWASP LLM06). |
| Multimodel / Multitenant | A pipeline invoking more than one AI model / shared infrastructure serving multiple isolated tenants. |
| Policy-as-code | Enforcement through machine-checkable policy (required checks, admission control) rather than manual attestation. |
| Poisoned pipeline execution | Unreviewed attacker-controlled pipeline config or code running with pipeline privileges (OWASP CICD-SEC-4). |
| Prompt injection | Crafted input an LLM treats as instruction, altering its behavior (OWASP LLM01). |
| Provenance | Verifiable evidence of how and from what an artifact was produced. |
| Risk tier | Tier 1–3 classification determining which enhanced controls apply beyond the baseline (Section 2.1). |
| SBOM | Software Bill of Materials: a formal inventory of a software artifact's components and dependencies. |
| SLSA Build L3 | A SLSA supply-chain level denoting non-falsifiable provenance from a hardened build service. |
| Workload identity | A federated, short-lived machine identity used to authenticate without static keys. |
| Zero Trust | A model that never trusts implicitly, verifies every request, enforces least privilege, and assumes breach (NIST SP 800-207). |

---

## Appendix C: References and Standards

Primary sources and frameworks underpinning this standard:

- [OWASP Top 10 CI/CD Security Risks](https://owasp.org/www-project-top-10-ci-cd-security-risks/)
- [OWASP Top 10 for LLM Applications (2025)](https://genai.owasp.org/llm-top-10/)
- [Microsoft Security Blog — Securing CI/CD in an agentic world (Claude Code GitHub Action case)](https://www.microsoft.com/en-us/security/blog/2026/06/05/securing-ci-cd-in-agentic-world-claude-code-github-action-case/)
- [Meta — Practical AI agent security (Agents Rule of Two)](https://ai.meta.com/blog/practical-ai-agent-security/)
- [Wiz — CI/CD security best practices](https://www.wiz.io/academy/application-security/ci-cd-security-best-practices)
- [OWASP CI/CD Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/CI_CD_Security_Cheat_Sheet.html)

Referenced by identifier throughout: NIST SP 800-53 Rev. 5; NIST SP 800-218 (SSDF v1.1) and SP 800-218A (generative-AI profile); NIST AI RMF (AI 100-1); NIST SP 800-207 (Zero Trust); NIST SP 800-161 and SLSA (supply chain); Microsoft Cloud Security Benchmark and the Entra, Key Vault, Defender for Cloud, and Sentinel documentation; MITRE ATLAS.

---

*This standard is a draft for review. Controls become binding on approval by the accountable owner named in the document-control table.*

**End of standard.**
