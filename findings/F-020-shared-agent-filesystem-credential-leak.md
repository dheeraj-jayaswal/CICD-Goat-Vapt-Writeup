# F-020: Shared Agent Filesystem Exposes Jenkins FreeStyle Job Credential (Twiddledum / flag6)

**SEVERITY: HIGH**

| Field | Detail |
|---|---|
| Affected Asset | Jenkins job `wonderland-twiddle`/`wonderland-twiddledum`, `agent1` filesystem (`/home/jenkins/.npmrc`) |
| OWASP CI/CD Mapping | **CICD-SEC-6** (Insufficient Credential Hygiene) — a credential written to a shared, persistent agent filesystem location is readable by any other job executing on the same agent · CICD-SEC-5 (Insufficient PBAC) — no isolation between jobs sharing an agent |
| Status | CONFIRMED, CTFd-verified (flag6) |

## Discovery

Twiddledum is a **FreeStyle** job (not a Pipeline job) — its build steps are configured directly in Jenkins' `config.xml`, not a Jenkinsfile. Reading its config via [CVE-2024-23897](./F-016-CVE-2024-23897-jenkins-cli-arbitrary-file-read.md) revealed that its build steps write a Jenkins-managed npm registry credential into `~/.npmrc` on `agent1` using a `withCredentials` binding — a standard pattern for npm authentication. Critically, the file is written to the agent's own home directory (`~/.npmrc`) rather than a job-scoped workspace, meaning it **persists between builds** and is readable by any subsequent job running on the same agent.

## Exploitation

Triggered the twiddledum job to force it to write the credential to disk, then immediately triggered our own `caterpillar` pipeline (which also runs on `agent1`) to simply read `~/.npmrc`:

```bash
# Step 1: trigger twiddledum to write the credential
curl -s -b $COOKIES -u alice:alice -X POST \
  "http://localhost:8080/job/wonderland-twiddle/job/wonderland-twiddledum/build"

# Step 2: caterpillar pipeline reads the file written by twiddledum
sh 'cat /home/jenkins/.npmrc'
# Output: //registry.npmjs.org/:_authToken=710866F2-2CED-4E60-A4EB-223FD892D95A
```

```
POST /api/v1/challenges/attempt {"challenge_id":6,"submission":"710866F2-2CED-4E60-A4EB-223FD892D95A"}
-> {"status":"correct"}
```

## Interview Talking Point

Agent isolation is a frequently-misunderstood boundary in CI/CD security. Multiple jobs sharing the same agent (even if they belong to different projects or teams) share the same filesystem, user home directory, network namespace, and process environment. Credentials written to home-directory dotfiles (`.npmrc`, `.gitconfig`, `.aws/credentials`), workspace artifacts not properly cleaned up, and `/tmp` files all persist across jobs and are accessible to any subsequent build. **True isolation requires ephemeral, per-job agents** (container-per-build or VM-per-build patterns) rather than a shared long-lived agent.
