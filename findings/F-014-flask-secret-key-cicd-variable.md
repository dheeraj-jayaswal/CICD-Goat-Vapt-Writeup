# F-014: Flask Session Secret Key Derived From a CI/CD Pipeline Variable

**SEVERITY: HIGH**

| Field | Detail |
|---|---|
| Affected Asset | `wonderland/nest-of-gold` (GitLab public project), `app.py` |
| OWASP CI/CD Mapping | **CICD-SEC-6** (Insufficient Credential Hygiene) — a CI/CD variable is reused as a production cryptographic secret with inconsistent values between environments |
| Related Web Vulnerability Class | OWASP Web Top 10 A02:2021 — Cryptographic Failures (predictable/leaked session-signing secret enables full session forgery) |
| Status | CONFIRMED at design/source-code level. Full exploitation (session forgery PoC) initially **blocked** — required either the live production value or reachability of the deployed instance, neither obtainable under black-box constraints. **Later completed** once [F-017](./F-017-gitlab-runner-token-secret-theft.md) recovered the real value. |

## Summary

GitLab's project API and Explore page permit anonymous listing of public-visibility projects without any authentication — this became a productive path after direct GitLab login/registration was ruled out (see [F-017](./F-017-gitlab-runner-token-secret-theft.md)).

```bash
curl -s "http://localhost:4000/api/v4/projects?visibility=public" \
  | jq -r '.[] | "\(.id) | \(.path_with_namespace)"'
curl -s "http://localhost:4000/api/v4/groups" | jq -r '.[] | "\(.id) | \(.full_path)"'
```

Result: 3 public projects across 2 groups — `wonderland/awesome-app`, `wonderland/nest-of-gold`, `pygryphon/pygryphon`.

Source review of `app.py` (pulled anonymously via GitLab's repository files API) revealed:

```python
import flask, flask_login, os
app = flask.Flask(__name__)
app.secret_key = os.getenv("FLAG11")

@app.route('/protected')
@flask_login.login_required
def protected():
    return 'Logged in as: ' + flask_login.current_user.id
```

Flask signs and validates all session cookies using `app.secret_key`. Any party who obtains this value can forge an arbitrarily-signed session cookie for **any user identity — including identities that do not exist in the application's own users dictionary** — achieving full authentication bypass, since `flask_login`'s cookie-based session does not re-check credentials against the users store once a signed session is present.

The vulnerability is that this value is sourced from a CI/CD pipeline variable (`FLAG11`) rather than being generated as a proper application secret. The pipeline definition shows two different values actively used across environments:

```yaml
# .gitlab-ci.yml - two different FLAG11 usages side by side
test-job:
  script:
    - export FLAG11=test   # hardcoded dummy value, test stage only

deploy-job:
  environment: production
  script:
    - docker run -d -e FLAG11=$FLAG11 -p 5000:5000 --name web web:latest
    # $FLAG11 here resolves to a GitLab CI/CD project variable - real value never observed at this point
```

This pattern — a single CI/CD variable name serving double duty as both a disposable test value and a production security boundary — is a direct example of CICD-SEC-6, and demonstrates concretely how a CI/CD hygiene issue becomes a full web application authentication bypass, not just a CI-internal information leak.

## What Would Complete This Finding (at time of initial discovery)

To move this from a confirmed design flaw to a fully weaponized PoC, either of the following would be required:

- Read access to the project's CI/CD variables (Settings > CI/CD > Variables), which requires Maintainer-level project membership or higher — blocked by the authentication wall.
- Alternatively, reachability of the deployed production instance (attempted at `localhost:5000` — connection refused).

## Resolution

This blocker was fully resolved by [F-017](./F-017-gitlab-runner-token-secret-theft.md) — registering a rogue GitLab CI runner and polling for jobs surfaced the real `FLAG11` value in plaintext (`7ED44218-C9CC-4824-BC85-C9841305A642`), since GitLab hands runners the real value of every CI/CD variable regardless of masked/protected settings. This is the exact value needed to forge a valid `nest-of-gold` session cookie.

## Remediation

- Never derive an application's cryptographic session-signing key from a CI/CD pipeline variable, especially one that is also assigned disposable/dummy values in non-production pipeline stages.
- Generate `secret_key` via a dedicated secrets manager (Vault, GitLab's own protected + masked CI/CD variables scoped strictly to the production environment, or a cloud KMS).
- Never allow a test-stage script to reference the same variable name used for a production secret.

## Interview Talking Points

> "While enumerating a second CI/CD platform in the same environment, I found a Flask application whose session `secret_key` was set directly from a CI/CD pipeline variable — and that same variable was assigned a hardcoded dummy value in the test stage of the pipeline. That's a real authentication-bypass risk, because anyone who learns that value can forge a valid session cookie for any user. I couldn't fully weaponize it immediately because I didn't have the access needed to read the real production value, but I documented it as a confirmed design flaw rather than dropping it, and was explicit about exactly what evidence was and wasn't obtained — and it later got fully resolved once a separate finding (F-017) recovered the actual value."

This is a good illustration of **why documenting a partially-blocked finding with precision matters**: rather than either overclaiming impact or abandoning it, stating exactly what's confirmed (design flaw) vs. unconfirmed (live exploitation) let a later, unrelated finding complete the chain.
