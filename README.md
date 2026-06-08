# Securing the CI/CD Pipeline in Azure

A security-architecture reference for securing a CI/CD pipeline where **source lives in GitHub** and **builds/deploys run on Azure DevOps Pipelines**. Written from a security-architect perspective: controls are justified by trust boundaries and threats, mapped to the Azure services that enable them, and sequenced against a recognized maturity model.

> **Scope:** Azure DevOps Services (cloud) and Azure DevOps Server (self-hosted)
> **Control sources:** Microsoft Learn (official documentation)
> **Risk taxonomy:** OWASP Top 10 CI/CD Security Risks (v1.0)
> **Maturity scale:** Azure Well-Architected Framework Security Maturity Model (Levels 1–5)

## What's inside

| File | Purpose |
|---|---|
| [`Securing-CICD-Pipeline-in-Azure.md`](./Securing-CICD-Pipeline-in-Azure.md) | The full reference document — design principles, reference architecture, threat model, layered control architecture, assurance, maturity roadmap, and residual-risk register. |
| `LICENSE` | License terms for this material. |
| `.gitignore` | Ignore rules for OS/editor cruft and optional docs-site build output. |

## How to use it

- **As a target-state reference** — adopt the layered controls (Section 5) and sequence them with the phased roadmap (Section 7).
- **As a gap assessment** — for an existing pipeline, score current state against the target tiers using the template in Appendix D, extended from the control-ID matrix in Appendix B.
- **As a control catalogue** — every control has a stable ID (e.g., `CTRL-IAM-01`) that threads the threat model, maturity model, roadmap, and risk register together.

## Document structure

1. Overview & Scope
2. Design Principles & Shared Responsibility
3. Reference Architecture & Solution Design
4. Assets & Threat Model
5. Defense-in-Depth Control Architecture
6. Assurance & Verification
7. Maturity Model & Roadmap
8. Trade-offs & Residual-Risk Register
   plus Appendices A–E (services inventory, traceability matrix, OWASP reference, gap-assessment template, sources & glossary)

## Sourcing & integrity notes

- Controls trace to official **Microsoft Learn** documentation; **OWASP Top 10 CI/CD Security Risks** is used solely as the risk-mapping taxonomy.
- **License-dependent** services (GitHub Advanced Security for Azure DevOps, Microsoft Defender for Cloud, Microsoft Sentinel / log ingestion) are flagged throughout.
- Known coverage gaps are recorded honestly in the **Residual-Risk Register** (Section 8) rather than overstated — notably artifact build-provenance (RR-03) and the absence of native audit streaming on Azure DevOps Server (RR-04).

## Publishing this repo

```bash
gh auth login
git init
git add .
git commit -m "Add CI/CD Azure security architecture reference"
gh repo create cicd-azure-security --public --source=. --push
```

## Disclaimer

This is a **target-state reference**, not a guarantee of compliance or security. Microsoft Learn documentation changes over time — verify specific control settings against the current official docs (linked in Appendix E) before implementation. This material is not affiliated with or endorsed by Microsoft or OWASP.

## License

Released under the terms in [`LICENSE`](./LICENSE). The default here is the MIT License for ease of reuse; if you intend this purely as documentation, you may prefer [Creative Commons Attribution 4.0 (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/) — swap the `LICENSE` file accordingly. Remember to set the copyright holder name in `LICENSE`.
