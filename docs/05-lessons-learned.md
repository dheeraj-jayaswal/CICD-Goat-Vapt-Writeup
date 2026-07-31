# Lessons Learned / Investigative Arcs Worth Remembering

A few cross-cutting patterns from this engagement that are easy to forget under time pressure, collected here separately from the [operational/technical notes](./01-methodology.md).

## 1. "This looks like a dead end" is not the same as "this is a dead end"
Mad Hatter and Cheshire Cat both initially looked like non-functional or unreachable jobs. Both turned out to hide CRITICAL findings once the platform's own diagnostic logs (branch-indexing logs, `config.xml` via CVE-2024-23897) were read directly instead of relying on UI-level behavior. **Read the platform's own logs before concluding "nothing here."**

## 2. A ruled-out conclusion should be re-checked against new evidence, not left standing
The Duchess/PyPI-token secret was dismissed as "organic open-source noise" — until learning that the commit author worked for the company that built the lab. The same underlying evidence didn't change; what changed was context that made the original interpretation much less likely. See [F-006](../findings/informational/F-006-private-repo-enumeration-gap.md) and the [Duchess correction](../ctfd/challenge-cross-reference.md#correction-duchess-flag4-likely-already-captured).

## 3. Question the mental model, not just the technique
Every GitLab authentication attempt (documented credentials, enumeration, self-registration) was correctly exhausted — but the real path ([F-017](../findings/F-017-gitlab-runner-token-secret-theft.md)) required no human login at all. The breakthrough came from questioning whether "compromise GitLab" meant "log in as a human," not from trying harder along the same path.

## 4. Masking and redaction are not access controls
Recurring across F-010, F-013, and F-017: Jenkins/GitLab both mask secret values in logs and UI output, but neither masking mechanism is designed to withstand an entity that is itself illegitimate (a shell step encoding the value before printing it; a rogue runner that must receive real values to function). **The actual security boundary for a secret is who/what can reach it, not whether it's displayed in cleartext.**

## 5. Know the difference between "blocked, here's precisely why" and "let me find another way in"
F-016's `hudson.util.Secret` binary-decryption barrier was correctly identified as a real, mathematically acknowledged dead end (per Jenkins' own security advisory) — rather than force-guessed indefinitely, a different, lower-effort path (a leftover plaintext bootstrap file) was pursued instead and succeeded. The Dormouse/flag9 investigation (6 confirmed dead ends) is the inverse case: exhausting every reasonable avenue and then honestly reporting the boundary held, which is itself a legitimate and valuable pentest outcome.

## 6. Positive controls deserve documentation too
[F-011](../findings/informational/F-011-correct-withcredentials-usage.md) documents a case where `withCredentials()` was used *correctly*, directly contrasting with the F-010 failure in the same environment. Documenting what worked, not just what broke, makes the report far more useful for remediation prioritization — it proves the platform is capable of the right behavior, so the fix is process, not a platform limitation.
