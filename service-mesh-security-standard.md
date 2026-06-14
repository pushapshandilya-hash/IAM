# Enterprise Service Mesh Security Standard

> Control-based security standard for service mesh implementations.

| Field | Value |
|---|---|
| **Document ID** | SEC-STD-SM-001 |
| **Version** | 2.1 |
| **Owner** | Enterprise Security Architecture |
| **Approved By** | Security Governance Board |
| **Classification** | Internal |
| **Review Cycle** | Annual |
| **Control Framework** | Internal `SM-*` control catalog (NIST 800-53 r5 / CSF 2.0 crosswalk in [Appendix B](#appendix-b-control-to-framework-reference); CI/CD controls also mapped to OWASP CI/CD Top 10) |
| **Derived From** | NIST SP 800-204 series and NIST SP 800-207 (Zero Trust) |

---

## Table of Contents

1. [Purpose and Scope](#1-purpose-and-scope)
2. [Definitions and Normative Language](#2-definitions-and-normative-language)
3. [Security Principles](#3-security-principles)
4. [Security Control Domains](#4-security-control-domains)
5. [Minimum Production Baseline](#5-minimum-production-baseline)
6. [Framework Compliance Mapping](#6-framework-compliance-mapping)
7. [Governance, Exceptions and Roles](#7-governance-exceptions-and-roles)
8. [References](#8-references)
- [Appendix A. Approved Technology Categories](#appendix-a-approved-technology-categories)
- [Appendix B. Control-to-Framework Reference](#appendix-b-control-to-framework-reference)
- [Appendix C. Deployment and Operational Guidance](#appendix-c-deployment-and-operational-guidance)
- [Document Control and Approval](#document-control-and-approval)

---

## 1. Purpose and Scope

This standard defines the mandatory security controls for all service mesh implementations, regardless of vendor or distribution. It is mesh-agnostic and applies to both sidecar and sidecar-less (ambient) data planes.

Each control is stated once with a unique ID. [Appendix B](#appendix-b-control-to-framework-reference) maps every control to NIST 800-53 Rev 5 and CSF 2.0 for audit traceability. Implementation guidance, approved products, and architecture diagrams live in the companion Service Mesh Security Design Guide (informative, not normative).

**In scope:**

- Kubernetes clusters, containerized workloads, microservices, and APIs governed by a service mesh
- Service mesh control planes and data planes
- Single-cluster, multi-cluster, and hybrid / multi-cloud mesh deployments
- Multitenant mesh deployments and tenant isolation
- Security of the CI/CD pipeline that builds and deploys mesh workloads and policy (pipeline security, not change-management process)

**Out of scope** (governed by separate enterprise standards, cross-referenced only):

- Incident response and change management processes

---

## 2. Definitions and Normative Language

This document uses MUST / SHOULD / MAY per RFC 2119. The **Crit.** column rates control importance: **Critical** blocks production authorization; **High** and **Medium** reflect descending risk (distinct from vulnerability severity in SM-VULN).

"Approved" technologies and authorities are those designated by Security Architecture in the applicable enterprise register.

| Term | Definition |
|---|---|
| **Control Plane** | Mesh components that configure and manage policy, identity, and certificates. |
| **Data Plane** | The components (sidecar proxies or node-level / ambient agents) that enforce policy and carry workload traffic. |
| **Workload Identity** | A unique, cryptographically verifiable identity bound to a workload (e.g., a SPIFFE SVID). |
| **Trust Domain / Trust Bundle** | A trust domain is the boundary within which identities share a common root of trust; a trust bundle is the set of CA certificates used to validate identities across domains. |
| **Gateway** | A dedicated proxy at the mesh edge: an ingress gateway handles inbound (north-south) traffic; an egress gateway brokers outbound traffic. |
| **mTLS Mode** | Mesh TLS posture: Permissive (plaintext + mTLS accepted) or Strict (mTLS only). |
| **East-West / North-South** | East-west is service-to-service traffic within or between clusters; north-south is ingress/egress traffic crossing the mesh boundary. |
| **Tenant / Multitenancy** | A tenant is a distinct consumer (team, application, business unit, or customer) sharing mesh infrastructure. Soft multitenancy isolates trusted tenants logically (namespaces); hard multitenancy isolates untrusted tenants physically (separate clusters or trust domains). |
| **Pipeline / Artifact / SBOM** | A pipeline is the automated CI/CD workflow that builds, tests, and deploys workloads and policy. An artifact is a build output (e.g., container image, Helm chart) promoted through it; an SBOM is the artifact's software bill of materials. |
| **Acronyms** | MFA (multi-factor authentication); SIEM (security information and event management); RBAC (role-based access control); PBAC (pipeline-based access controls); PKI (public-key infrastructure); HSM/KMS (hardware security module / key management service); mTLS (mutual TLS); NTP (network time protocol); SAST/DAST (static / dynamic application security testing); SCA (software composition analysis); IaC (infrastructure as code). |

---

## 3. Security Principles

All controls derive from four principles:

- **Zero Trust** — no workload, user, namespace, or network segment is trusted by default.
- **Least Privilege** — access is limited to the minimum required for the business function.
- **Explicit Verification** — every request is authenticated and authorized before access is granted.
- **Assume Breach** — controls are designed assuming an attacker may already hold a workload or network segment.

---

## 4. Security Control Domains

Controls are grouped by domain. Each ID follows the scheme `SM-<DOMAIN>-<NN>` and is the stable reference for exceptions, audit findings, and GRC tracking. Framework mappings for every control are listed in [Appendix B](#appendix-b-control-to-framework-reference).

### 4.1 Identity and Authentication (SM-IAM)

Every workload and administrator must carry a strong, verifiable identity.

| ID | Control Requirement | Crit. |
|---|---|---|
| **SM-IAM-01** | Every workload MUST have a unique cryptographic identity (SPIFFE-aligned). Shared workload identities are prohibited. | Critical |
| **SM-IAM-02** | Workload identities and credentials MUST be automatically provisioned and short-lived; no static or long-lived credentials. | Critical |
| **SM-IAM-03** | Administrative access MUST use the enterprise identity provider with MFA. | Critical |
| **SM-IAM-04** | External user authentication MUST use an approved federation protocol (OIDC, OAuth 2.0, or SAML 2.0). | Critical |
| **SM-IAM-05** | In multi-cluster or federated meshes, cross-cluster identity MUST use a shared trust domain or validated federated trust bundles. | High |

**Prohibited:** shared service accounts · hardcoded credentials · static API keys · long-lived tokens

### 4.2 Authorization (SM-AUTHZ)

Access between services is explicit, scoped, and deny-by-default.

| ID | Control Requirement | Crit. |
|---|---|---|
| **SM-AUTHZ-01** | Service-to-service authorization MUST be deny-by-default; only explicit allow rules permit communication. | Critical |
| **SM-AUTHZ-02** | Authorization policies MUST be workload-specific. Wildcard and allow-all policies are prohibited. | Critical |
| **SM-AUTHZ-03** | Namespace isolation MUST be enforced. Cluster-wide administrative access for application workloads is prohibited. | Critical |
| **SM-AUTHZ-04** | Administrative access MUST be separated from operational access (mesh and Kubernetes RBAC). | High |
| **SM-AUTHZ-05** | Ingress gateways MUST enforce request-level authentication and authorization for external traffic (e.g., token / JWT validation) before routing to workloads. | High |

> **Note:** where multiple tenants share a cluster, the tenancy model (soft or hard isolation) MUST be documented and approved.

### 4.3 Encryption in Transit / mTLS (SM-MTLS)

All service traffic is mutually authenticated and encrypted.

| ID | Control Requirement | Crit. |
|---|---|---|
| **SM-MTLS-01** | All production service-to-service traffic MUST use Strict mutual TLS. | Critical |
| **SM-MTLS-02** | Both client and server identities MUST be validated on every connection. | Critical |
| **SM-MTLS-03** | Plaintext MUST NOT traverse the network between workloads in production; gateway traffic MUST use TLS. (Application-to-local-proxy loopback is out of scope.) | Critical |

**mTLS posture by environment:**

| Environment | mTLS Requirement |
|---|---|
| Development | Permissive allowed |
| Test | Strict preferred |
| Staging | Strict required |
| Production | Strict mandatory |

> Exceptions require Security Architecture approval, a risk assessment, compensating controls, and a remediation date (see [§7](#7-governance-exceptions-and-roles)).

### 4.4 Certificate and Key Management (SM-CERT)

Identity certificates are issued, rotated, and protected automatically.

| ID | Control Requirement | Crit. |
|---|---|---|
| **SM-CERT-01** | Certificates MUST be issued only from an approved Certificate Authority ([Appendix A](#appendix-a-approved-technology-categories)). | Critical |
| **SM-CERT-02** | Certificate issuance, renewal, and rotation MUST be automated and non-disruptive to running services. | Critical |
| **SM-CERT-03** | Certificate lifetimes MUST NOT exceed the defined maximums. | Critical |
| **SM-CERT-04** | Private keys MUST be protected by approved controls (HSM / KMS) and certificate expiry MUST be monitored. | Critical |
| **SM-CERT-05** | A certificate revocation and CA-compromise response posture MUST be defined; short-lived certificates MAY substitute for active revocation, and CA compromise MUST trigger trust-bundle rotation. | High |

**Maximum certificate lifetimes:**

| Certificate Type | Maximum Lifetime |
|---|---|
| Root CA | 15 years |
| Intermediate CA | 3 years |
| Mesh CA | 12 months |
| Workload Certificate | 24 hours |

> **Note:** short-lived certificate validation depends on synchronized time; workloads MUST use a trusted time source (NTP).

### 4.5 Network Segmentation (SM-NET)

Mesh policy supplements — never replaces — network-layer controls.

| ID | Control Requirement | Crit. |
|---|---|---|
| **SM-NET-01** | Kubernetes Network Policies MUST be implemented in addition to mesh authorization policy. | High |
| **SM-NET-02** | Production workloads MUST be isolated and East-West traffic restricted to approved flows. | High |
| **SM-NET-03** | Traffic between meshed and non-meshed (legacy) workloads MUST cross a controlled, authenticated boundary; non-meshed workloads MUST NOT bypass mesh authorization. | High |

### 4.6 Egress Control (SM-EGR)

Outbound traffic is brokered, restricted, and logged.

| ID | Control Requirement | Crit. |
|---|---|---|
| **SM-EGR-01** | Outbound traffic MUST route through an approved egress gateway; direct internet egress from workloads is prohibited. | High |
| **SM-EGR-02** | Egress destinations MUST be explicitly approved and all egress traffic MUST be logged. | High |

### 4.7 Secrets Management (SM-SEC)

Secrets live in managed stores, never in code or images.

| ID | Control Requirement | Crit. |
|---|---|---|
| **SM-SEC-01** | Secrets MUST be stored in an approved secrets platform and encrypted at rest and in transit. | Critical |
| **SM-SEC-02** | Secrets MUST be rotated on a defined schedule and all secret access MUST be audited. | Critical |

**Prohibited:** secrets in source code · secrets in container images · secrets in environment variables · shared credentials

### 4.8 Logging (SM-LOG)

Security-relevant mesh events are captured and centralized.

| ID | Control Requirement | Crit. |
|---|---|---|
| **SM-LOG-01** | Authentication, authorization decisions, certificate lifecycle, gateway, administrative, policy-change, and security-violation events MUST be logged. | High |
| **SM-LOG-02** | All mesh security telemetry MUST be integrated with the enterprise SIEM. | High |
| **SM-LOG-03** | Logs MUST be retained per the enterprise schedule (production 365 days; critical systems 7 years). | Medium |
| **SM-LOG-04** | Security logs MUST be protected against tampering and unauthorized access (integrity and access controls). | High |

### 4.9 Monitoring and Detection (SM-MON)

Posture and anomalies are continuously monitored and alerted.

| ID | Control Requirement | Crit. |
|---|---|---|
| **SM-MON-01** | Monitoring MUST cover mTLS compliance, policy violations, certificate expiry, lateral-movement, and anomalous traffic. | High |
| **SM-MON-02** | Alerts MUST be configured at the defined severities and routed to the SOC. | High |

**Minimum alert set:**

| Alert Condition | Severity |
|---|---|
| Certificate expiry < 30 days | High |
| mTLS failure | Critical |
| Unauthorized communication | Critical |
| Policy modification | High |
| Control-plane failure | Critical |
| Gateway failure | Critical |
| Excessive authorization denials | High |

### 4.10 Policy-as-Code and Enforcement (SM-PAC)

Policy is version-controlled, reviewed, and enforced at admission.

| ID | Control Requirement | Crit. |
|---|---|---|
| **SM-PAC-01** | All mesh and network policies MUST be stored in source control; manual production changes are prohibited. | High |
| **SM-PAC-02** | All policy changes MUST pass peer review and automated security validation before deployment. | High |
| **SM-PAC-03** | Policies MUST be enforced at admission (admission controller) to block non-compliant deployments. | High |

### 4.11 Secure Configuration (SM-CFG)

Control plane, data plane, and gateways ship hardened.

| ID | Control Requirement | Crit. |
|---|---|---|
| **SM-CFG-01** | The control plane MUST be deployed highly-available, with TLS enabled, restricted administrative access, and audit logging. | High |
| **SM-CFG-02** | Data-plane attachment MUST be automatic (sidecar injection or ambient enrollment); proxies MUST run patched images and set resource limits. | High |
| **SM-CFG-03** | Gateways MUST use TLS 1.2 as a minimum and SHOULD use TLS 1.3, with cipher suites per the enterprise cryptographic standard, strong certificate validation, and access logging. | High |
| **SM-CFG-04** | Control-plane-to-data-plane configuration channels MUST be mutually authenticated and encrypted. | High |

### 4.12 Vulnerability Management (SM-VULN)

Mesh components are scanned and remediated within SLA.

| ID | Control Requirement | Crit. |
|---|---|---|
| **SM-VULN-01** | Mesh components (control plane, sidecars, gateways, certificate services) MUST be in scope for monthly vulnerability scanning. | High |
| **SM-VULN-02** | Vulnerabilities MUST be remediated within SLA by finding severity: Critical 7 days, High 30 days, Medium 90 days. | High |

### 4.13 Multitenancy and Isolation (SM-MT)

A tenant is any group sharing the mesh — a team, application, business unit, or customer. Match isolation to trust: trusted tenants can share a cluster behind logical fences (namespaces and policy); untrusted or regulated tenants need hard separation (their own cluster or mesh). Use the lightest isolation that fits the trust level.

| ID | Control Requirement | Crit. |
|---|---|---|
| **SM-MT-01** | The tenancy model (soft or hard) MUST be documented and approved per cluster, with the isolation boundary (namespace, cluster, or mesh trust domain) explicitly identified and matched to the tenant trust level. | High |
| **SM-MT-02** | Namespaces provide logical isolation only; each tenant's workloads, identities, policies, and quotas MUST be namespace-scoped, with cross-namespace traffic deny-by-default. | Critical |
| **SM-MT-03** | Soft-tenancy namespaces MUST enforce the full isolation set: RBAC (who can act), Network Policy (L3/L4), mesh Authorization Policy (L7, identity-based), and Pod Security admission (workload hardening). | Critical |
| **SM-MT-04** | Per-tenant resource quotas and limits MUST be enforced to prevent resource exhaustion and noisy-neighbor abuse. | High |
| **SM-MT-05** | Untrusted, hostile, or regulated tenants MUST use hard isolation (separate cluster or mesh trust domain); a shared mesh control plane MUST serve only tenants of equivalent trust. | High |
| **SM-MT-06** | Tenant configuration MUST be standardized and automated (namespace templates and per-tenant policy-as-code) for consistent, drift-free isolation. | High |

> **Note:** namespaces give logical isolation, not hard isolation — they assume the shared node, kernel, and control plane are trustworthy. For untrusted or hostile tenants, namespaces are not enough; escalate to a separate cluster or mesh trust domain.


---

## 5. Minimum Production Baseline

Every production service mesh deployment MUST satisfy the following baseline. Failure to meet any item constitutes non-compliance and requires a formal, time-boxed risk acceptance (see [§7](#7-governance-exceptions-and-roles)).

| Baseline Requirement | Setting | Controls |
|---|---|---|
| Workload identity | Unique, SPIFFE-aligned | SM-IAM-01/02 |
| Authorization | Deny-by-default | SM-AUTHZ-01/02 |
| mTLS | Strict | SM-MTLS-01 |
| Certificate rotation | Automated | SM-CERT-02 |
| Namespace isolation | Enforced | SM-AUTHZ-03 |
| Network policies | Required | SM-NET-01 |
| Egress control | Gateway-brokered | SM-EGR-01 |
| Secrets management | Managed store | SM-SEC-01 |
| Logging + SIEM | Integrated | SM-LOG-01/02 |
| Policy-as-Code | Source-controlled + admission-enforced | SM-PAC-01/03 |
| Multitenancy | Documented isolation model | SM-MT-01/02 |
| Pipeline security | Signed artifacts + scoped creds + stage gates | SM-CICD-02/06/09 |
| Security reviews | Quarterly | SM-GOV (see §7) |

---

## 6. Framework Compliance Mapping

Framework-level mappings (PCI-DSS, SOC 2, ISO 27001, HIPAA) are maintained in the enterprise GRC platform. Control-level NIST 800-53 / CSF traceability is provided in [Appendix B](#appendix-b-control-to-framework-reference).

---

## 7. Governance, Exceptions and Roles

### 7.1 Security Reviews

| Review | Frequency |
|---|---|
| RBAC / authorization review | Quarterly |
| Certificate review | Quarterly |
| Mesh security review | Quarterly |
| Penetration testing | Annual |
| Architecture review | Annual |

### 7.2 Exceptions

Any deviation from a control MUST reference the specific control ID and be recorded in the enterprise GRC platform. Each exception MUST include:

- Business justification and risk assessment
- Compensating controls
- An expiration date
- Approval by Security Architecture, Risk Management, and the Application Owner

> Exceptions exceeding 12 months require executive approval.

### 7.3 Roles and Responsibilities

| Role | Responsibility |
|---|---|
| Security Architecture | Owns and maintains this standard; approves exceptions |
| Platform Engineering | Implements and operates mesh controls |
| Application Teams | Ensure workload compliance with the baseline |
| PKI Team | Operate certificate authorities and key protection |
| SOC | Monitor mesh security events and alerts |
| Internal Audit | Validate compliance against control IDs |
| Risk Management | Track and report approved exceptions |

### 7.4 Compliance Validation

Compliance is validated through automated scanning, quarterly control reviews, architecture assessments, internal audits, and annual penetration testing. Evidence is retained per enterprise requirements.

---

## 8. References

Authoritative sources. Links verified current; items with no free public source are marked.

| Reference | Version | Source |
|---|---|---|
| NIST SP 800-204 / 204A / 204B / 204C | 2019–21 | <https://csrc.nist.gov/pubs/sp/800/204/final> |
| NIST SP 800-207 / 207A (Zero Trust; 207A = service mesh) | 2020 / 23 | <https://csrc.nist.gov/pubs/sp/800/207/final> |
| NIST SP 800-53 (current release 5.2.0) | Rev 5 | <https://csrc.nist.gov/pubs/sp/800/53/r5/final> |
| NIST Cybersecurity Framework | 2.0 (2024) | <https://www.nist.gov/cyberframework> |
| CIS Kubernetes Benchmark | Current | <https://www.cisecurity.org/benchmark/kubernetes> |
| CNCF Cloud Native Security Whitepaper | Current | <https://tag-security.cncf.io/community/resources/security-whitepaper/> |
| OWASP Kubernetes Top 10 | 2022 | <https://owasp.org/www-project-kubernetes-top-ten/> |
| OWASP Top 10 CI/CD Security Risks | v1.0 (2022) | <https://owasp.org/www-project-top-10-ci-cd-security-risks/> |
| OWASP DevSecOps Guideline | Current | <https://owasp.org/www-project-devsecops-guideline/> |
| DORA (DevOps delivery metrics) | Current | <https://dora.dev/> |
| Wiz — CI/CD Security Best Practices | 2026 | <https://www.wiz.io/academy/application-security/ci-cd-security-best-practices> |
| PCI DSS | v4.0.1 | <https://www.pcisecuritystandards.org/document_library/> |
| ISO/IEC 27001 | 2022 | *No free public source (ISO, paywalled)* |
| SOC 2 (AICPA Trust Services Criteria) | TSC 2017/22 | *No free public source (AICPA)* |
| HIPAA Security Rule (45 CFR Part 164) | Current | <https://www.hhs.gov/hipaa/for-professionals/security/index.html> |
| Companion: Design Guide; Incident Response; Change Management | — | *Internal documents* |

---

## Appendix A. Approved Technology Categories

*Informative. Capability classes, not product endorsements. The named, version-specific product register is maintained by Security Architecture in the companion Design Guide and updated independently of this standard's review cycle.*

| Capability | Requirement |
|---|---|
| Certificate authority | Enterprise-approved PKI or certificate-management service issuing automated, short-lived workload certificates |
| Secrets management | Centrally-managed, audited secrets platform with encryption at rest and supported rotation |
| Federation protocols | Open standards only: OpenID Connect (OIDC), OAuth 2.0, SAML 2.0 |
| Policy enforcement | Admission-control mechanism integrated with CI/CD policy validation |
| Workload identity | Open workload-identity standard providing unique, verifiable, short-lived credentials |
| Pipeline security scanning | SAST, SCA, IaC, container, and DAST scanners integrated as blocking pipeline gates |
| Artifact integrity | Artifact / image signing with admission-time signature verification |

---

## Appendix B. Control-to-Framework Reference

Normative crosswalk. Each `SM-*` control maps to its supporting NIST SP 800-53 Rev 5 control(s) and NIST CSF 2.0 category, for audit and exception traceability.

| Control ID | NIST SP 800-53 Rev 5 | NIST CSF 2.0 |
|---|---|---|
| SM-IAM-01 | IA-2, IA-5, IA-9 | PR.AA-01, PR.AA-02 |
| SM-IAM-02 | IA-5, IA-5(2) | PR.AA-01 |
| SM-IAM-03 | IA-2(1), IA-2(2) | PR.AA-03 |
| SM-IAM-04 | IA-8 | PR.AA-03 |
| SM-IAM-05 | IA-9, SC-8 | PR.AA-01 |
| SM-AUTHZ-01 | AC-3, AC-4 | PR.AA-05 |
| SM-AUTHZ-02 | AC-3, AC-6 | PR.AA-05 |
| SM-AUTHZ-03 | AC-6, SC-7 | PR.AA-05 |
| SM-AUTHZ-04 | AC-5, AC-6 | PR.AA-05 |
| SM-AUTHZ-05 | AC-3, IA-8 | PR.AA-05 |
| SM-MTLS-01 | SC-8, SC-8(1), SC-13 | PR.DS-02 |
| SM-MTLS-02 | IA-3, SC-8 | PR.DS-02 |
| SM-MTLS-03 | SC-8, SC-23 | PR.DS-02 |
| SM-CERT-01 | SC-12, SC-17 | PR.DS-02 |
| SM-CERT-02 | SC-12, IA-5(2) | PR.AA-01 |
| SM-CERT-03 | SC-12 | PR.DS-02 |
| SM-CERT-04 | SC-12, SC-28 | PR.DS-01 |
| SM-CERT-05 | SC-12, SC-17 | PR.AA-01 |
| SM-NET-01 | SC-7, AC-4 | PR.IR-01 |
| SM-NET-02 | SC-7, AC-4 | PR.IR-01 |
| SM-NET-03 | SC-7, AC-4 | PR.IR-01 |
| SM-EGR-01 | SC-7(4), SC-7(8) | PR.IR-01 |
| SM-EGR-02 | SC-7, AU-2 | DE.CM-01 |
| SM-SEC-01 | SC-12, SC-28 | PR.DS-01 |
| SM-SEC-02 | IA-5, AU-2 | PR.AA-01 |
| SM-LOG-01 | AU-2, AU-3, AU-12 | DE.CM-01, DE.AE-03 |
| SM-LOG-02 | AU-6, SI-4 | DE.CM-01 |
| SM-LOG-03 | AU-11 | DE.AE-03 |
| SM-LOG-04 | AU-9 | PR.PS-01 |
| SM-MON-01 | SI-4, AU-6 | DE.CM-01, DE.AE-02 |
| SM-MON-02 | SI-4 | DE.AE-02 |
| SM-PAC-01 | CM-3, CM-5 | PR.PS-01 |
| SM-PAC-02 | CM-3, CM-4 | PR.PS-01 |
| SM-PAC-03 | CM-7, AC-3 | PR.PS-01 |
| SM-CFG-01 | CM-6, SC-8, AU-2 | PR.PS-01 |
| SM-CFG-02 | CM-6, SI-2 | PR.PS-01, PR.PS-02 |
| SM-CFG-03 | SC-8, SC-13, AU-2 | PR.DS-02 |
| SM-CFG-04 | SC-8, IA-3 | PR.DS-02 |
| SM-VULN-01 | RA-5, SI-2 | ID.RA-01 |
| SM-VULN-02 | SI-2, RA-5 | ID.RA-01 |
| SM-MT-01 | AC-4, CM-2 | PR.IR-01 |
| SM-MT-02 | AC-4, SC-7 | PR.IR-01, PR.AA-05 |
| SM-MT-03 | AC-3, AC-6, SC-7 | PR.AA-05, PR.IR-01 |
| SM-MT-04 | SC-6 | PR.IR-01 |
| SM-MT-05 | SC-7, CM-2 | PR.IR-01 |
| SM-MT-06 | CM-2, CM-6 | PR.PS-01 |
| SM-CICD-01 | CM-3, SA-15 | PR.PS-01 |
| SM-CICD-02 | AC-6, IA-5 | PR.AA-05 |
| SM-CICD-03 | SR-3, RA-5 | GV.SC, ID.RA-01 |
| SM-CICD-04 | CM-7, SC-7 | PR.PS-01 |
| SM-CICD-05 | IA-5, SC-12 | PR.AA-01 |
| SM-CICD-06 | SI-7, CM-14 | PR.PS-01 |
| SM-CICD-07 | SR-3, SA-9 | GV.SC |
| SM-CICD-08 | AU-2, SI-4 | DE.CM-01 |
| SM-CICD-09 | SA-11, RA-5 | ID.RA-01 |

**CI/CD controls — OWASP Top 10 CI/CD Security Risks mapping:**

| Control | OWASP CI/CD Risk |
|---|---|
| SM-CICD-01 | CICD-SEC-1 (Insufficient Flow Control) |
| SM-CICD-02 | CICD-SEC-2 / 5 (IAM; Insufficient PBAC) |
| SM-CICD-03 | CICD-SEC-3 (Dependency Chain Abuse) |
| SM-CICD-04 | CICD-SEC-4 / 7 (Poisoned Pipeline Execution; Insecure Config) |
| SM-CICD-05 | CICD-SEC-6 (Insufficient Credential Hygiene) |
| SM-CICD-06 | CICD-SEC-9 (Improper Artifact Integrity Validation) |
| SM-CICD-07 | CICD-SEC-8 (Ungoverned 3rd-Party Services) |
| SM-CICD-08 | CICD-SEC-10 (Insufficient Logging & Visibility) |
| SM-CICD-09 | CICD-SEC-1 / 3 / 4 (enforced via stage gates) |

---

## Appendix C. Deployment and Operational Guidance

*Informative. This annex provides non-normative guidance for applying the multitenancy (SM-MT) and CI/CD (SM-CICD) controls. It does not add requirements.*

### C.1 Multitenancy Boundary Model

Isolation strength increases from logical (namespace) to physical (cluster) to identity (trust domain). Choose the weakest boundary that meets the tenant trust level.

| Boundary | Isolation | Typical use |
|---|---|---|
| Namespace | Soft / logical | Trusted tenants sharing a cluster and mesh |
| Cluster | Hard / physical | Untrusted tenants; strict blast-radius limits |
| Mesh trust domain | Hard / identity | Cross-cluster or federated isolation |

*Recommended baseline:* namespace-per-tenant with Network Policy, mesh Authorization Policy, RBAC, and resource quotas as the soft-isolation default; escalate to cluster or trust-domain separation for untrusted or regulated tenants.

### C.2 Tenancy and Deployment-Model Mapping

| Deployment model | Isolation | When to use |
|---|---|---|
| Shared cluster, shared mesh | Soft (namespace) | Trusted internal teams |
| Shared cluster, multiple meshes | Medium | Stronger control-plane separation |
| Cluster per tenant | Hard | Untrusted or regulated tenants |
| Federated multi-cluster | Hard (trust domains) | Geographic / business-unit isolation at scale 

---

## Document Control and Approval

| Version | Date | Author / Note |
|---|---|---|
| 1.0 | YYYY-MM-DD | Security Architecture — control-catalog baseline |


**Approval signatures:**

| Role | Name | Signature | Date |
|---|---|---|---|
| Security Architect | | | |
| Platform Engineering Lead | | | |
| Head of Security | | | |
| Risk Manager | | | |
| Governance Board Chair | | | |
