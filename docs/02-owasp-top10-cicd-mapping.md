# OWASP Top 10 CI/CD Security Risks — Reference

Provided as a standalone reference, since every finding in this repo is mapped against it. This is the [OWASP Top 10 CI/CD Security Risks (2023)](https://owasp.org/www-project-top-10-ci-cd-security-risks/) list — worth memorizing given how frequently CI/CD security comes up in AppSec/DevSecOps discussions and interviews.

| Code | Risk | One-line description | Findings in this repo |
|---|---|---|---|
| **CICD-SEC-1** | Insufficient Flow Control Mechanisms | Pipeline can proceed / merge / deploy without adequate gating, approval, or verification of what's actually changing. | [F-013](../findings/F-013-insecure-automerge-bypass.md), [F-021](../findings/F-021-checkov-sast-config-override-bypass.md) |
| **CICD-SEC-2** | Inadequate Identity and Access Management | Overly broad, shared, or unscoped identities/tokens across the pipeline; lack of per-system, per-repo least privilege. | [F-010](../findings/F-010-jenkins-secrets-exposure-console-logs.md), [F-017](../findings/F-017-gitlab-runner-token-secret-theft.md) |
| **CICD-SEC-3** | Dependency Chain Abuse | Untrusted or compromised upstream dependencies (packages, base images, plugins) are pulled into the build unverified. | *(not directly exploited in this engagement)* |
| **CICD-SEC-4** | Poisoned Pipeline Execution (PPE) | Untrusted input (PR branches, forked repos, webhook payloads) can control or influence pipeline execution. | [F-010](../findings/F-010-jenkins-secrets-exposure-console-logs.md), [F-018](../findings/F-018-decoupled-pipeline-branch-exclusion-bypass.md), [F-019](../findings/F-019-jenkins-controller-rce-agent-label-override.md) |
| **CICD-SEC-5** | Insufficient Pipeline-Based Access Controls (PBAC) | Pipelines/jobs have more access to systems, credentials, or environments than the task requires. | [F-013](../findings/F-013-insecure-automerge-bypass.md), [F-019](../findings/F-019-jenkins-controller-rce-agent-label-override.md), [F-020](../findings/F-020-shared-agent-filesystem-credential-leak.md) |
| **CICD-SEC-6** | Insufficient Credential Hygiene | Secrets are stored, transmitted, or logged in ways that make them recoverable (plaintext env dumps, console logs, hardcoded values). | [F-010](../findings/F-010-jenkins-secrets-exposure-console-logs.md), [F-014](../findings/F-014-flask-secret-key-cicd-variable.md), [F-017](../findings/F-017-gitlab-runner-token-secret-theft.md), [F-018](../findings/F-018-decoupled-pipeline-branch-exclusion-bypass.md), [F-020](../findings/F-020-shared-agent-filesystem-credential-leak.md) |
| **CICD-SEC-7** | Insecure System Configuration | The CI/CD platform itself (Jenkins, GitLab, etc.) is misconfigured — weak defaults, unnecessary exposure, missing hardening. | [F-016](../findings/F-016-CVE-2024-23897-jenkins-cli-arbitrary-file-read.md) |
| **CICD-SEC-8** | Ungoverned Usage of 3rd Party Services | Unvetted integrations/plugins/services granted access into the pipeline without governance. | [F-021](../findings/F-021-checkov-sast-config-override-bypass.md) |
| **CICD-SEC-9** | Improper Artifact Integrity Validation | Build artifacts are not signed/verified before deployment, allowing tampering between build and deploy. | *(not directly exploited in this engagement)* |
| **CICD-SEC-10** | Insufficient Logging and Visibility | Pipeline activity (builds, merges, credential use) is not logged/monitored well enough to detect abuse. | Discussed as a contributing/audit-trail factor in [F-013](../findings/F-013-insecure-automerge-bypass.md) |

## How to use this table

Each finding file states its primary and contributing OWASP CI/CD mapping in its metadata table up top. Use this page as the index if you want to study or present findings **by risk category** rather than by finding ID — e.g., "show me every example of Insufficient Credential Hygiene in this lab" maps to CICD-SEC-6's row above.
