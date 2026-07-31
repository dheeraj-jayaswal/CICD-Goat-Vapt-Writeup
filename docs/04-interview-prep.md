# Interview-Ready Summary

One-paragraph summaries for each major finding, phrased the way you'd open an answer to "walk me through a vulnerability you found."

## F-010 — Jenkins Console Log Secrets Exposure
> "I found that a Jenkins pipeline triggered by pull requests had a debug stage that dumped the full environment to the build console, leaking two live Gitea tokens. I validated both were real, confirmed one had push access to a separate repository, and proved full impact by pushing an unauthorized commit with the stolen token and showing Jenkins automatically built it — so it was a complete PR-to-RCE chain, not just a theoretical leak."

## F-013 — Insecure Auto-Merge Bypass
> "I found a pipeline that auto-merged pull requests it judged to be 'just a version bump,' but the check only verified the diff's word count balanced to zero across the whole PR, not per file. I proved that by modifying the version file and simultaneously smuggling an unrelated change into another file, keeping the total word delta at zero — the PR auto-merged with zero human review, and I confirmed the smuggled content went live on main."

## F-014 — Flask Secret Key from a CI/CD Variable
> "While enumerating a second CI/CD platform in the same environment, I found a Flask application whose session secret_key was set directly from a CI/CD pipeline variable — and that same variable was assigned a hardcoded dummy value in the test stage of the pipeline. That's a real authentication-bypass risk, because anyone who learns that value can forge a valid session cookie for any user. I couldn't fully weaponize it in this engagement because I didn't have the access needed to read the real production value or reach the deployed instance, but I documented it as a confirmed design flaw rather than dropping it, and I was explicit in my report about exactly what evidence was and wasn't obtained."

## On hitting an access-control wall (GitLab authentication)
> "I tried the provided credentials, tested for username enumeration, and attempted self-registration — each ruled out for a specific, confirmed technical reason, not abandoned on assumption. When I couldn't get past email confirmation with no mail server available, I called that a legitimate result: the access control held. I still found a real finding through anonymous source review alone, which is worth emphasizing — authentication isn't always a prerequisite for meaningful findings, and knowing when to stop pushing on a dead end versus when to keep digging is itself a skill worth demonstrating."

## F-016 — CVE-2024-23897 (Jenkins CLI Arbitrary File Read)
> "While hunting for one specific credential, I hit a wall where the target account had almost no permissions at all. Rather than keep guessing at RBAC bypasses, I checked whether the exact Jenkins version had any known CVEs, found CVE-2024-23897 — a real, critical arbitrary file read bug in the Jenkins CLI — and used it to read files directly off the Jenkins controller with an account that had almost no privileges. I hit a genuine, documented dead end trying to decrypt one specific binary secret, correctly identified why it was mathematically infeasible rather than forcing it, and found the actual flag a different way: a leftover plaintext bootstrap file the CVE let me read just as easily."

## F-017 — GitLab Runner Token
> "After exhausting every reasonable user-authentication path against GitLab and confirming each was genuinely blocked, I stepped back and questioned the assumption itself — the objective was to compromise GitLab, not necessarily to log into it. I found the instance's shared-runner registration token was a default value documented publicly, used it to register my own rogue CI runner with two API calls, and then simply asked GitLab for pending jobs. GitLab handed me every CI/CD variable in plaintext, including ones marked masked and protected, because the runner genuinely needs the real values to execute a job. That's how I got the flag, and it also fully weaponized an earlier finding where I only had a theoretical vulnerability but not the actual secret value."

## Common follow-up questions to be ready for

- **"Why didn't masking/redaction stop you in F-010/F-013/F-017?"** — Masking is a *log-redaction* control, not an *access* control. It prevents a secret from appearing in a UI/log; it does nothing to stop an entity that legitimately needs the real value (a shell step, a CI runner) from obtaining it, or a determined party from encoding around the mask pattern (e.g. base64-piping the value before it's printed).
- **"How is F-016 different from the rest of your findings?"** — Every other finding here is *design/logic abuse* of correctly-functioning features. F-016 is a genuine, numbered software vulnerability (a real CVE) in unpatched code. Be ready to work both angles in an interview: business-logic flaws *and* vulnerability research against known CVEs.
- **"What do you do when a finding can't be fully weaponized?"** (see F-014) — Document it as a confirmed *design-level* flaw with an explicit statement of exactly what would be needed to complete exploitation, rather than either overclaiming impact or dropping it because it isn't a full PoC.
- **"How do you find business-logic flaws in a pipeline, not just technical bugs?"** — Read the pipeline definition (Jenkinsfile / `.gitlab-ci.yml`) directly rather than only black-box probing. Several findings here (F-013, F-018, F-021) were only fully understood by reading the actual shell/YAML logic, not by observing build behavior alone.
