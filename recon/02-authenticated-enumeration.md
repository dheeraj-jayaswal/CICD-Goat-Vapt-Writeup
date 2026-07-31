# Phase 2 — Authenticated Enumeration

Goal: use provided grey-box credentials (`alice/alice`) to fully enumerate Jenkins job structure, cross-reference against Gitea's visible repositories, and identify gaps worth investigating.

## Jenkins Job Enumeration

```bash
JENKINS_CRUMB=$(curl -s -u alice:alice "http://localhost:8080/crumbIssuer/api/json" | jq -r '.crumb')
curl -s -u alice:alice http://localhost:8080/api/json | jq .
curl -s -u alice:alice http://localhost:8080/computer/api/json | jq .
curl -s -u alice:alice http://localhost:8080/asynchPeople/api/json | jq .
```

Crumb explanation: `crumbIssuer` returns Jenkins' CSRF token, required for any state-changing (POST) request — fetched proactively even though the initial calls here are read-only GETs.

**Result:** 9 Jenkins jobs discovered — `wonderland-caterpillar-prod`, `wonderland-caterpillar-test`, `wonderland-cheshire-cat`, `wonderland-dodo` (folder), `wonderland-dormouse`, `wonderland-mad-hatter`, `wonderland-mock-turtle`, `wonderland-twiddle` (folder), `wonderland-white-rabbit`. Node enumeration revealed two executors: the built-in controller node and one labeled agent (`agent1`/`myagent`). User enumeration revealed three accounts: `admin`, `alice`, and a previously-unknown account, `knave` (see [F-007](../findings/informational/F-007-additional-jenkins-user.md)).

## Job-to-Repository Mismatch (F-006)

Cross-referencing the 9 Jenkins jobs against the 7 publicly visible Gitea repositories revealed 4 jobs with no visible matching repo: `wonderland-cheshire-cat`, `wonderland-mad-hatter`, `wonderland-mock-turtle`, `wonderland-white-rabbit`.

This was fully resolved later (see [F-010](../findings/F-010-jenkins-secrets-exposure-console-logs.md)) once a leaked Jenkins-held Gitea token was used to enumerate its own repo access, confirming these four repos are simply **private**, reachable only by specific service accounts. Full writeup: [`findings/informational/F-006-private-repo-enumeration-gap.md`](../findings/informational/F-006-private-repo-enumeration-gap.md).

## Multibranch Project Navigation

Several jobs (white-rabbit, mad-hatter, cheshire-cat, mock-turtle, dormouse) are Jenkins **Multibranch Pipeline** projects — the top-level job is a container; actual buildable jobs exist one level down, per-branch or per-pull-request.

```bash
# Enumerating branches / PRs under a multibranch project
curl -s -u alice:alice "http://localhost:8080/job/wonderland-white-rabbit/api/json" \
  | jq -r '.jobs[]? | "\(.name) | \(.url)"'
```

**Result:** white-rabbit exposed only `PR-1` and `PR-2` (no mainline branch job) — a design signal in itself, since PR-only pipelines mean every reachable build is triggered by external, potentially untrusted contribution (directly relevant to CICD-SEC-4, Poisoned Pipeline Execution).

**Recurring operational issue:** requests to `config.xml` on these jobs initially returned HTTP 403 even though the same `alice` account could list job names via the top-level API. This is Jenkins' distinct `Job/Read` vs `Job/ExtendedRead` permission grants (see [F-009](../findings/informational/F-009-job-read-vs-extendedread.md)) — `alice` had the former but not the latter. Workaround used throughout: read build results via `lastBuild/wfapi/describe` and `lastBuild/consoleText` instead of the raw `config.xml`, both of which only require `Job/Read`.
