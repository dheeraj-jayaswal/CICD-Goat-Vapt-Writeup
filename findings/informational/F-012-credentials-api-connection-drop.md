# F-012: Credentials Store API Returns Connection Drop Instead of Standard 403

**SEVERITY: INFO**

Querying Jenkins' credentials store API directly resulted in a silent connection drop (`curl: 'Empty reply from server'`) rather than the standard 403 HTML error page seen everywhere else in this engagement:

```bash
curl -v -u alice:alice "http://localhost:8080/credentials/store/system/domain/_/api/json?depth=1"
# ... Empty reply from server
```

This suggests a filter-level block on this specific path, distinct from Jenkins' normal RBAC 403 response path — not exploitable, but a notable defensive posture worth contrasting against [F-009](./F-009-job-read-vs-extendedread.md)'s more typical 403 behavior.
