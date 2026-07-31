# F-011: Correct Use of withCredentials() (Positive Control Example)

**SEVERITY: INFO — Positive Example**

The `caterpillar` Jenkinsfile's deploy stage used Jenkins' `withCredentials()` binding correctly, masking the token value in console output:

```
[Pipeline] withCredentials
Masking supported pattern matches of $TOKEN
+ curl -isSL http://wonderland:1234/api/user -H Authorization: Token **** -H Content-Type: application/json
```

Valuable as a direct, same-environment contrast to [F-010](../F-010-jenkins-secrets-exposure-console-logs.md): this proves the platform is fully capable of safe credential handling when `withCredentials()` is used as intended — the vulnerability in F-010 was a design/process failure (a raw `env` dump bypassing this mechanism entirely), not a platform limitation. Useful interview point: **"the tool wasn't broken, the pipeline authoring practice was."**

This also surfaced an unexplored lead: an internal service at `wonderland:1234/api/user` referenced by this stage, which failed to resolve in this context (DNS resolution failure) — likely belongs to a different challenge's Docker network and was not pursued further within this engagement's scope.
