# F-007: Additional Jenkins User Account Discovered

**SEVERITY: LOW**

Jenkins' `asynchPeople/api/json` endpoint revealed a third user account beyond `admin` and `alice`: `knave`.

```bash
curl -s -u alice:alice http://localhost:8080/asynchPeople/api/json | jq .
```

Not exploited during this assessment (no credential brute-force was performed, consistent with the agreed rules of engagement), but noted as a candidate for credential-testing in a follow-up session, and as a thematically consistent detail (Alice-in-Wonderland naming convention used throughout the lab).
