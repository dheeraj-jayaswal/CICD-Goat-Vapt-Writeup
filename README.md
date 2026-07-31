# CICD-Goat VAPT Writeup

> A full vulnerability assessment & penetration test against [OWASP CICD-Goat](https://github.com/cider-security-research/cicd-goat) — a deliberately vulnerable CI/CD environment (Jenkins, Gitea, GitLab, CTFd) — mapped end-to-end against the **OWASP Top 10 CI/CD Security Risks**.

[![Made for OWASP CICD-Goat](https://img.shields.io/badge/target-OWASP%20CICD--Goat-blue)](https://github.com/cider-security-research/cicd-goat)
[![Findings](https://img.shields.io/badge/findings-16-critical)](./findings)
[![Flags Captured](https://img.shields.io/badge/CTFd%20flags-9%2F11-success)](./ctfd/challenge-cross-reference.md)
[![License: MIT](https://img.shields.io/badge/license-MIT-lightgrey)](./LICENSE)

---

## Why this exists

Most CI/CD security writeups either stay purely theoretical (a slide explaining "poisoned pipeline execution") or purely CTF-flag-chasing (a one-line "here's the flag, next"). This repo tries to do neither: every finding below is a **fully validated, PoC-backed vulnerability**, mapped to a specific [OWASP CI/CD-SEC](./docs/02-owasp-top10-cicd-mapping.md) risk category, written the way you'd actually want to explain it in an interview or a real client report — including the dead ends, the wrong assumptions, and how they got corrected.

If you're studying for an AppSec/DevSecOps/CI-CD-security interview, prepping for a pentest engagement involving a CI/CD toolchain, or just want a concrete, hands-on tour of what "Poisoned Pipeline Execution" or "Insufficient Credential Hygiene" actually looks like on the wire — this is written for you.

## Target environment

| Field | Value |
|---|---|
| Target | [OWASP CICD-Goat](https://github.com/cider-security-research/cicd-goat) — local Docker Compose deployment |
| Engagement type | Authorized self-directed learning lab (grey-box) |
| Tech stack | Jenkins 2.332.1, Gitea 1.16.5, GitLab 15.11.13-ee, CTFd, Docker Compose |
| Scope | `localhost:3000` (Gitea), `:8080`/`:50000` (Jenkins), `:4000` (GitLab), `:8000` (CTFd), `:8008` (prod-sim) |
| Methodology | Phase 0–3 (Scope → Fingerprinting → Vulnerability ID → Exploitation) |

Full rules of engagement: [`docs/00-engagement-overview.md`](./docs/00-engagement-overview.md).

> ⚠️ **All testing in this repo was performed against the tester's own local, disposable, intentionally-vulnerable Docker Compose lab.** No production systems, shared infrastructure, or third-party data were involved. Spin up your own copy of CICD-Goat from the [official repo](https://github.com/cider-security-research/cicd-goat) before trying any of this yourself.

## Results at a glance

| ID | Title | Severity | OWASP CI/CD Mapping | CTFd Flag |
|---|---|---|---|---|
| [F-010](./findings/F-010-jenkins-secrets-exposure-console-logs.md) | Secrets exposure in Jenkins build console logs → credential theft → unauthorized repo write | **CRITICAL** | CICD-SEC-6, -4, -2 | flag1, flag2 |
| [F-013](./findings/F-013-insecure-automerge-bypass.md) | Insecure auto-merge logic bypasses code review (PR-wide word-diff heuristic) | **CRITICAL** | CICD-SEC-1, -5 | flag10 |
| [F-016](./findings/F-016-CVE-2024-23897-jenkins-cli-arbitrary-file-read.md) | CVE-2024-23897 — Jenkins CLI arbitrary file read on the controller | **CRITICAL** | CICD-SEC-7 | flag8 |
| [F-017](./findings/F-017-gitlab-runner-token-secret-theft.md) | GitLab shared-runner registration token → instance-wide CI/CD secret theft | **CRITICAL** | CICD-SEC-2, -6 | flag11 |
| [F-019](./findings/F-019-jenkins-controller-rce-agent-label-override.md) | Jenkins controller-node code execution via agent label override | **CRITICAL** | CICD-SEC-5, -4 | flag5 |
| [F-018](./findings/F-018-decoupled-pipeline-branch-exclusion-bypass.md) | Decoupled pipeline repo + branch exclusion filter bypass | HIGH | CICD-SEC-4, -6 | flag3 |
| [F-020](./findings/F-020-shared-agent-filesystem-credential-leak.md) | Shared agent filesystem exposes FreeStyle job credential | HIGH | CICD-SEC-6, -5 | flag6 |
| [F-021](./findings/F-021-checkov-sast-config-override-bypass.md) | Checkov SAST config override enables undetected IaC misconfiguration | HIGH | CICD-SEC-1, -8 | flag7 |
| [F-014](./findings/F-014-flask-secret-key-cicd-variable.md) | Flask session secret key derived from a CI/CD pipeline variable | HIGH | CICD-SEC-6 | flag11 (via F-017) |

Plus **6 informational / supporting findings** (positive controls, RBAC boundary confirmations, minor info-disclosure) in [`findings/informational/`](./findings/informational).

### How the findings chain together

Several findings aren't independent — one directly enables or completes another. This is the part that tends to impress in an interview more than any single finding on its own:

![CICD-Goat cross-finding kill chain diagram showing F-010 leading to unauthorized push, F-013's auto-merge bypass, F-016's CVE-2024-23897 arbitrary file read feeding flag captures and F-018, and F-017's rogue GitLab runner completing F-014](./docs/assets/kill-chain-diagram.svg)

**9 of 11 CTFd challenges solved and flag-verified** — see the full [challenge cross-reference](./ctfd/challenge-cross-reference.md), including an honestly-documented case (Dormouse/flag9) where the access-control boundary held under sustained attack.

## Repo structure

```
.
├── docs/                          # Engagement context, methodology, mappings, remediation, interview prep
│   ├── 00-engagement-overview.md
│   ├── 01-methodology.md
│   ├── 02-owasp-top10-cicd-mapping.md
│   ├── 03-remediation-roadmap.md
│   ├── 04-interview-prep.md
│   └── 05-lessons-learned.md
├── recon/                         # Phase 1 & 2 — fingerprinting and authenticated enumeration
│   ├── 01-fingerprinting.md
│   └── 02-authenticated-enumeration.md
├── findings/                      # One file per confirmed finding, full PoC + remediation
│   ├── F-010-...md ... F-021-...md
│   └── informational/             # INFO/LOW severity supporting observations
├── ctfd/
│   └── challenge-cross-reference.md
└── LICENSE
```

## Reading paths

- **Just want the highlights?** Start with the [results table above](#results-at-a-glance), then read F-010, F-013, F-016, and F-017 — the four most complete end-to-end kill chains.
- **Studying for an interview?** Go straight to [`docs/04-interview-prep.md`](./docs/04-interview-prep.md) — one-paragraph, spoken-style summaries of every major finding, plus common follow-up questions.
- **Building/hardening a CI/CD pipeline?** Go straight to [`docs/03-remediation-roadmap.md`](./docs/03-remediation-roadmap.md) — a prioritized, actionable checklist.
- **New to CI/CD security concepts?** Start with [`docs/02-owasp-top10-cicd-mapping.md`](./docs/02-owasp-top10-cicd-mapping.md) for the reference taxonomy this whole repo is organized around.

## About OWASP Top 10 CI/CD Security Risks

Every finding here is mapped against the [OWASP Top 10 CI/CD Security Risks (2023)](https://owasp.org/www-project-top-10-ci-cd-security-risks/) — a full reference table (CICD-SEC-1 through CICD-SEC-10) lives in [`docs/02-owasp-top10-cicd-mapping.md`](./docs/02-owasp-top10-cicd-mapping.md), since it's referenced constantly throughout the individual findings.

## Disclaimer

This repository documents testing performed **exclusively against a local, self-hosted, intentionally-vulnerable training lab** ([OWASP CICD-Goat](https://github.com/cider-security-research/cicd-goat)), for educational and portfolio purposes. Nothing here targets, references, or was tested against any production system, third-party service, or real credential. Do not use any technique in this repo against systems you do not own or have explicit written authorization to test.

## License

Content in this repository is licensed under the [MIT License](./LICENSE). CICD-Goat itself is a separate project by [Cider Security](https://github.com/cider-security-research/cicd-goat) — go star the original.

## Author

**Dheeraj Kumar Jayaswal** — Certified Ethical Hacker, Web Application & API Penetration Tester.

Feedback, corrections, and PRs (e.g. for flag9/Dormouse, or the Duchess/flag4 follow-up in the [CTFd cross-reference](./ctfd/challenge-cross-reference.md)) are welcome — see [CONTRIBUTING.md](./CONTRIBUTING.md).
