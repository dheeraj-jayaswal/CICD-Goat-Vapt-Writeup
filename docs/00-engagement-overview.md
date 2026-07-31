# Engagement Overview

## Rules of Engagement

| Field | Value |
|---|---|
| Target name | CICD-Goat (OWASP) — local Docker Compose deployment |
| Platform | Lab (self-hosted, localhost) |
| Engagement type | Learning Lab / Authorized self-directed VAPT |
| Industry vertical | N/A — generic CI/CD toolchain (Jenkins, Gitea, GitLab, CTFd) |
| In-scope assets | `localhost:3000` (Gitea), `:8080`/`:50000` (Jenkins), `:4000` (GitLab), `:8000` (CTFd), `:8008` (prod-sim) |
| Out of scope | Docker host OS, local LAN, anything outside the 'goat' compose network |
| Credentials provided | Yes — grey-box: `alice/alice` (Jenkins/CTFd), `thealice/thealice` (Gitea), `alice/alice1234` (GitLab) |
| Testing style | Grey-box, credentials used from the outset |
| Tech stack | Jenkins 2.332.1 (Jetty 9.4.43), Gitea 1.16.5 (Go), GitLab (nginx), CTFd (gunicorn), Docker Compose |
| Restrictions | None beyond stated scope — full technique freedom, destructive testing permitted (disposable lab) |

## Objective

Perform a full black-box-then-grey-box VAPT sweep against a locally hosted, intentionally vulnerable CI/CD environment (OWASP CICD-Goat), with primary focus on CI/CD-specific attack classes (as opposed to generic web application vulnerabilities), and produce fully validated, PoC-backed findings mapped to the OWASP Top 10 CI/CD Security Risks — for both remediation guidance and interview/reference use.

## Methodology

Testing followed a phased approach consistent with standard bug-bounty / VAPT methodology, adapted for a local CI/CD-focused lab:

- **Phase 0** — Rules of Engagement & Scope Confirmation
- **Phase 1** — Active Fingerprinting (service identification, tech stack enumeration)
- **Phase 2** — Vulnerability Identification (authenticated enumeration, secrets discovery, pipeline analysis)
- **Phase 3** — Exploitation (live PoCs, credential validation, impact confirmation)

See [`01-methodology.md`](./01-methodology.md) for the detailed operational approach and lessons learned.

All commands referenced throughout this repository were executed against the tester's own local Docker Compose deployment. No production systems, shared infrastructure, or third-party data were involved at any point.

## Executive Summary

This assessment identified multiple fully validated **CRITICAL**-severity vulnerabilities in the CICD-Goat lab's Jenkins/Gitea/GitLab CI/CD toolchain, each independently exploited end-to-end with working proof-of-concept exploits. Findings map directly to distinct categories in the [OWASP Top 10 CI/CD Security Risks (2023)](./02-owasp-top10-cicd-mapping.md), making this environment a strong practical reference for each vulnerability class.

See the [results table in the main README](../README.md#results-at-a-glance) for the full findings list, and [`ctfd/challenge-cross-reference.md`](../ctfd/challenge-cross-reference.md) for how findings map to CICD-Goat's own scored challenges.

### Key Risk Themes

- Credential hygiene failures in build logs are the single highest-value target in this environment — a debug-oriented "dump environment variables" pattern directly leaked usable, scoped-but-real credentials.
- Automated CI/CD trust decisions (auto-merge, auto-deploy) based on shallow heuristics (diff shape, filename, word count) rather than semantic content review are exploitable and should never bypass mandatory human review or branch protection.
- PR-triggered pipelines (Poisoned Pipeline Execution surface) meaningfully increase blast radius, since low-privilege or even anonymous external contributors can reach pipeline execution simply by opening a pull request.
- Where least-privilege was actually applied (e.g., per-repo credential scoping), it measurably reduced blast radius — a useful contrast point for remediation guidance.
