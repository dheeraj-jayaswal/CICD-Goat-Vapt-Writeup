# Contributing

This repo documents a personal VAPT engagement against [OWASP CICD-Goat](https://github.com/cider-security-research/cicd-goat). Contributions are welcome, especially:

- **Corrections** to any technical detail, command, or CVE reference.
- **Completions** of the two open items: the Dormouse/flag9 investigation ([details](./ctfd/challenge-cross-reference.md#dormouse-flag9--thoroughly-investigated-honestly-blocked)) and the Duchess/flag4 confirmation ([details](./ctfd/challenge-cross-reference.md#correction-duchess-flag4-likely-already-captured)).
- **Additional interview talking points** or alternative exploitation paths for any existing finding.
- **Translations** of the writeup (open an issue first to coordinate structure).

## Ground rules

- Keep all findings scoped to the OWASP CICD-Goat lab (or an equivalent disposable, self-hosted training environment) — this repo does not document or link to real-world/production exploitation.
- Cite the specific OWASP CI/CD-SEC category(ies) for any new or modified finding.
- Follow the existing finding template (see any file under [`findings/`](./findings)) for consistency: Summary → Discovery → Exploitation → Impact → Remediation → Interview Talking Points.
- Open an issue before a large restructuring PR so it can be discussed first.

## Reporting an issue

Use the [bug report template](./.github/ISSUE_TEMPLATE/correction.md) for factual corrections, or just open a plain issue for anything else.
