# Consolidated Remediation Roadmap

Prioritized action list across every finding in this engagement. Use this as a checklist when hardening a real CI/CD toolchain, not just this lab.

| Priority | Action | Addresses |
|---|---|---|
| 1 — Immediate | Rotate any tokens leaked via build console logs (e.g. per-repo and generic service account tokens) | [F-010](../findings/F-010-jenkins-secrets-exposure-console-logs.md) |
| 1 — Immediate | Remove or redact any debug "dump environment variables" pipeline stage | [F-010](../findings/F-010-jenkins-secrets-exposure-console-logs.md) |
| 1 — Immediate | Enable branch protection + required reviewers on all repos with CI auto-merge capability | [F-013](../findings/F-013-insecure-automerge-bypass.md) |
| 1 — Immediate | Upgrade Jenkins to 2.442+ / LTS 2.426.3+, or disable CLI access as an interim mitigation | [F-016 — CVE-2024-23897](../findings/F-016-CVE-2024-23897-jenkins-cli-arbitrary-file-read.md) |
| 1 — Immediate | Rotate GitLab's shared-runner registration token; prefer scoped project/group runners over shared runners | [F-017](../findings/F-017-gitlab-runner-token-secret-theft.md) |
| 2 — Short-term | Rework auto-merge/auto-deploy gates to verify per-file semantic content, never PR-wide diff-shape aggregates | [F-013](../findings/F-013-insecure-automerge-bypass.md) |
| 2 — Short-term | Require manual approval for first-time/untrusted PR contributors before pipelines build their code | [F-010](../findings/F-010-jenkins-secrets-exposure-console-logs.md) |
| 2 — Short-term | Stop deriving application secrets (session keys, etc.) from CI/CD pipeline variables; use a dedicated secrets manager | [F-014](../findings/F-014-flask-secret-key-cicd-variable.md) |
| 2 — Short-term | Restrict pipelines to dedicated agent nodes explicitly (avoid `agent any`); disable execution on the controller/built-in node | [F-019](../findings/F-019-jenkins-controller-rce-agent-label-override.md) |
| 2 — Short-term | Ensure SAST/IaC-scanner config files (`.checkov.yaml`, `.tflint.hcl`, etc.) are not modifiable by the same contributor whose code they scan | [F-021](../findings/F-021-checkov-sast-config-override-bypass.md) |
| 3 — Ongoing | Scope every CI credential to the minimum single repo/purpose required | [F-010](../findings/F-010-jenkins-secrets-exposure-console-logs.md) and general IAM hygiene |
| 3 — Ongoing | Adopt automated secret-scanning of build console **output**, not just source code | [F-010](../findings/F-010-jenkins-secrets-exposure-console-logs.md) |
| 3 — Ongoing | Audit logging/alerting on every automated merge event, tied to the human PR author, not just the bot identity | [F-013](../findings/F-013-insecure-automerge-bypass.md) |
| 3 — Ongoing | Require authentication for container registry repository/tag listing even on public projects | [F-015](../findings/informational/F-015-anonymous-registry-enumeration.md) |
| 3 — Ongoing | Move to ephemeral, per-job agents (container-per-build / VM-per-build) instead of long-lived shared agents | [F-020](../findings/F-020-shared-agent-filesystem-credential-leak.md) |
| 3 — Ongoing | Monitor and alert on new CI runner/agent registrations, particularly with unfamiliar descriptions or tags | [F-017](../findings/F-017-gitlab-runner-token-secret-theft.md) |

## Cross-cutting principle

Nearly every CRITICAL finding in this engagement traces back to one of two root patterns:

1. **A secret was handled by a mechanism that assumes the receiving party is legitimate** (masked variables, `withCredentials()` bindings, protected CI variables) — none of these are a substitute for controlling **who or what can register as, or execute as, that receiving party in the first place** (F-010, F-013, F-017).
2. **An automated trust decision (auto-merge, SAST gate, branch discovery filter) was satisfied by a shallow, checkable property (diff shape, file presence, config value) rather than genuine semantic verification** (F-013, F-021, F-018).

Fixing these two patterns generically — not just patching each individual finding — closes most of the attack surface documented in this repo.
