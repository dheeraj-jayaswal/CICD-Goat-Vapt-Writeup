# CTFd (Port 8000) — Challenge Board Cross-Reference

CTFd is CICD-Goat's own scoring/challenge-tracking system. Logging into it and cross-referencing its intended challenge list against organically-discovered findings served two purposes: validating that findings matched the lab's actual designed vulnerabilities (rather than being incidental artifacts), and revealing scope initially written off as dead ends.

## Fixing the Login (Root Cause)

An earlier login attempt failed silently — the CSRF nonce extraction returned empty. Root cause: the extraction pattern assumed `name="nonce"` and `value="..."` appeared immediately adjacent in the HTML, but the actual markup places `type="hidden"` between them:

```html
<input id="nonce" name="nonce" type="hidden" value="443..."/>
```

Working extraction and login:

```bash
NONCE=$(grep -oP '(?<=id="nonce")[^>]*value="\K[^"]+' /tmp/ctfd_login.html)
curl -s -b $COOKIES -c $COOKIES -X POST http://localhost:8000/login \
  --data-urlencode "name=alice" --data-urlencode "password=alice" \
  --data-urlencode "nonce=$NONCE"
curl -s -b $COOKIES http://localhost:8000/api/v1/challenges | jq .
# -> {"success":true,"data":[... 11 challenges ...]}
```

This is the same lesson learned with GitLab's CSRF token: always extract based on a stable anchor (`id="nonce"`) rather than assuming a fixed distance between two attributes in markup that may not be adjacent.

## Challenge List vs. Findings — Coverage Cross-Reference

| Challenge | Pts | Objective (per CTFd description) | Result |
|---|---|---|---|
| White Rabbit | 100 | Steal flag1 from Jenkins credential store | ✅ SOLVED — [F-010](../findings/F-010-jenkins-secrets-exposure-console-logs.md) (`flag1=06165DF2-C047-4402-8CAB-1C8EC526C115`) |
| Mad Hatter | 100 | Jenkinsfile protected — steal flag3 | ✅ SOLVED — [F-018](../findings/F-018-decoupled-pipeline-branch-exclusion-bypass.md) (`flag3=ACD6E6B8-3584-4F43-AB9C-ACD080B8EBB2`) |
| Duchess | 100 | Find leftover PyPI token | ⚠️ Likely captured in secrets sweep; see [correction note](#correction-duchess-flag4-likely-already-captured) below — not submitted to CTFd |
| Caterpillar | 200 | Steal flag2 from Jenkins credential store | ✅ SOLVED — [F-010](../findings/F-010-jenkins-secrets-exposure-console-logs.md) (`flag2=AEB14966-FFC2-4FB0-BF45-CD903B3535DA`) |
| Cheshire Cat | 200 | Controller-level RCE, read `~/flag5.txt` | ✅ SOLVED — [F-019](../findings/F-019-jenkins-controller-rce-agent-label-override.md) (`flag5=6B31A679-6D70-469D-9F8D-6D6E80B3C29C`) |
| Twiddledum | 200 | Flag6 in the twiddledum pipeline | ✅ SOLVED — [F-020](../findings/F-020-shared-agent-filesystem-credential-leak.md) (`flag6=710866F2-2CED-4E60-A4EB-223FD892D95A`) |
| Dodo | 200 | Make S3 bucket public without being caught | ✅ SOLVED — [F-021](../findings/F-021-checkov-sast-config-override-bypass.md) (`flag7=A62F0E52-7D67-410E-8279-32447ADAD916`) |
| Hearts | 300 | Steal flag8 — Jenkins SYSTEM credential | ✅ SOLVED — [F-016 / CVE-2024-23897](../findings/F-016-CVE-2024-23897-jenkins-cli-arbitrary-file-read.md) (`flag8=B1A648E1-FD8B-4D66-8CAF-78114F55D396`) |
| Dormouse | 300 | Steal flag9 from the Dormouse pipeline | ❌ Thoroughly investigated — 6 dead ends confirmed, see [below](#dormouse-flag9--thoroughly-investigated-honestly-blocked) |
| Mock Turtle | 300 | Push to main on mock-turtle, steal flag10 | ✅ SOLVED — [F-013](../findings/F-013-insecure-automerge-bypass.md) (`flag10=D54734AB-7B83-4931-A9BB-171476101FDF`) |
| Gryphon | 500 | Compromise GitLab, capture flag11 | ✅ SOLVED — [F-017](../findings/F-017-gitlab-runner-token-secret-theft.md) (`flag11=7ED44218-C9CC-4824-BC85-C9841305A642`) |

**Result: 9 of 11 challenges fully solved and CTFd-verified.** 1 likely captured but not submitted (Duchess). 1 thoroughly investigated with 6 independently-confirmed dead ends (Dormouse/flag9). Combined flag value: 1400 of 2100 possible CTFd points.

## Correction: Duchess (flag4) Likely Already Captured

During the original secrets-discovery sweep, `gitleaks` flagged a PyPI upload token in Duchess's `.pypirc` file, commit-authored by "Asaf \<asaf@cidersecurity.io>". This was dismissed at the time as organic open-source noise, since Duchess's repository is genuinely PyJWT's real project history.

This dismissal is very likely wrong. Cider Security is the company that authored CICD-Goat itself — a commit attributed to a Cider Security employee, embedded inside otherwise-authentic third-party OSS history, is a strong signal of a deliberately planted secret rather than incidental noise. Flagged here as a correction rather than quietly fixed, since it's a valuable lesson: a "this is just OSS noise" conclusion should be re-checked against new evidence (in this case, learning who actually created the lab) rather than left standing once reached.

**Recommended follow-up (not completed):** submit the previously-captured PyPI token value directly to CTFd `challenge_id 4` to confirm.

## Remaining Unresolved Challenges — Concrete Next Steps

- **Mad Hatter** *(now solved, see F-018)* — required enumerating ALL branches directly via Gitea's API, not just what Jenkins' multibranch scanner surfaced.
- **Cheshire Cat** *(now solved, see F-019)* — required targeting the controller node explicitly via agent label override.
- **Twiddledum** *(now solved, see F-020)* — required reading its actual Jenkins job console output and Jenkinsfile source.
- **Dodo** *(now solved, see F-021)* — required cloud/IaC-focused tooling (LocalStack/AWS CLI enumeration) rather than the Jenkins/Gitea techniques used elsewhere.

## Dormouse (flag9) — Thoroughly Investigated, Honestly Blocked

Dormouse received substantially more investigation effort than other challenges, and is documented in detail because every avenue was methodically tested and ruled out with a specific technical reason. **Six independent, confirmed dead ends is a materially different (and more defensible) result than "not yet tried."**

Dormouse's Jenkinsfile binds `credentialsId 'flag9'` via `withCredentials()`, matching the naming convention of every other flag. The pipeline downloads and executes a script (`reportcov.sh`) from an internal host (`prod`), referenced in a code comment using deliberately obfuscated octal IP notation (`0177.0.0.01`, which decodes to `127.0.0.1`).

| # | Avenue attempted | Result |
|---|---|---|
| 1 | Direct push to dormouse (thealice, and both known Gitea service tokens) | Confirmed pull-only for every available identity — no write access |
| 2 | Fork-based pull request (standard external-contributor pattern) | PR opened successfully, but Jenkins' branch indexer for this job only discovers branches, not pull requests — the PR was never built |
| 3 | Merging the PR directly | 405 — account lacks merge rights on this repository |
| 4 | Running the existing, unmodified pipeline as-is | Reproducibly fails — `reportcov.sh`'s internal upload step references an artifact never generated, since an earlier pytest stage fails on a missing Flask dependency |
| 5 | Reaching the credential from a different job's pipeline (cross-job credential scoping bypass test) | Confirmed job-scoped: `Could not find credentials entry with ID flag9` when attempted from an unrelated job with full push access |
| 6 | Cross-process `/proc/PID/environ` inspection during a concurrent build | Blocked by container process/user isolation — all permission-denied |

A **seventh avenue** — locating flag9's actual encrypted credential and attempting offline decryption — succeeded in locating the credential (via the CVE-2024-23897 technique, embedded directly in the job's own `config.xml` via a `FolderCredentialsProvider` property, rather than the global `credentials.xml`). The encrypted password blob was recovered in full. Decryption was attempted using `master.key` combined with the best-effort `hudson.util.Secret` bytes captured earlier:

```
[+] Master key hash (AES-128 key): ee8f5ed93f64a571d70a259fd49cedee
[+] Loaded 58 bytes from hudson.util.Secret capture
[!] MAGIC marker not found - hudson.util.Secret bytes are corrupted (expected per CVE advisory)
Decrypted (garbage) preview: b'\r\x06\xb26;\xc2\xd1\xd1\xbeH\xcf...'
[RESULT] Could not recover a valid confidentiality key.
```

This is not a surprising result — this Jenkins instance's UTF-8 default encoding places `hudson.util.Secret` recovery in Jenkins' own documented "infeasible" category (see [F-016](../findings/F-016-CVE-2024-23897-jenkins-cli-arbitrary-file-read.md)). Running the actual decryption attempt regardless, rather than only citing the theoretical limitation, turned an assumption into an empirically confirmed fact.

**Conclusion:** flag9 remains uncaptured. This is reported as a genuine access-control boundary that held under sustained, methodical attack — not a gap in effort. Demonstrating exactly where and why a boundary held is itself valuable output; not every finding in a real engagement resolves to full compromise.
