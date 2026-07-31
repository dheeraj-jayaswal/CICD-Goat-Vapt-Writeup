# F-009: Jenkins Job/Read vs Job/ExtendedRead Authorization Boundary

**SEVERITY: INFO**

`alice`'s account could list job existence and metadata via `/api/json` (`Job/Read`) but consistently received 403 when requesting a job's raw `config.xml` (`Job/ExtendedRead`) — two genuinely distinct Jenkins RBAC permissions.

```bash
curl -s -u alice:alice -w "%{http_code}\n" http://localhost:8080/job/wonderland-white-rabbit/config.xml
# -> HTTP 403, while /job/wonderland-white-rabbit/api/json returns 200
```

This is expected, correctly functioning least-privilege behavior, not a vulnerability — documented because it shaped methodology: pipeline content had to be read via `lastBuild/wfapi/describe`, `lastBuild/consoleText`, and direct Gitea source review instead of the more obvious `config.xml` endpoint.
