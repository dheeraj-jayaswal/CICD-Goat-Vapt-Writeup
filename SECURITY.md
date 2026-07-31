# Security Policy

## About This Repository

This repository documents a full VAPT engagement against [OWASP CICD-Goat](https://github.com/cider-security-research/cicd-goat), a deliberately vulnerable CI/CD training lab. It contains findings, PoCs, and remediation guidance written for educational and portfolio purposes. It does not contain a running service, real infrastructure, or any production system reference — every technique documented here was performed against a local, disposable, self-hosted lab, as stated in the repository's disclaimer.

## Reporting a Security Concern

If you find something in this repository that could be a genuine security concern, please report it privately rather than opening a public issue. This includes:

- Any real, working credential, API key, or secret that isn't clearly part of the disposable CICD-Goat lab environment (all lab credentials referenced here are specific to that self-hosted training environment and are not real production values)
- Any real, personally identifiable information (names, phone numbers, emails, addresses) that shouldn't be public
- Any content that could be mistaken for a working exploit against a real, currently-running target rather than the intentionally vulnerable training lab
- Any broken or misleading link that could redirect to a malicious destination

**To report:** Please open a private security advisory via this repository's **Security** tab ("Report a vulnerability"), or reach out directly via [LinkedIn](https://linkedin.com/in/dheerajkumarjayaswal).

Please do not open a public GitHub issue for sensitive findings — this gives time to review and fix before the detail is public.

## What This Policy Does Not Cover

- Disagreements about a finding's severity, CVE mapping, or technical detail (open a regular issue or PR per [CONTRIBUTING.md](./CONTRIBUTING.md) — happy to hear these)
- The open Dormouse/flag9 investigation or Duchess/flag4 confirmation (tracked separately, see [CONTRIBUTING.md](./CONTRIBUTING.md))
- General questions about the material

## Response

This is a personally maintained repository, not a funded project with an SLA. I'll do my best to respond and address anything genuine as quickly as I reasonably can.

---

Thank you for helping keep this resource accurate and safe to use.
