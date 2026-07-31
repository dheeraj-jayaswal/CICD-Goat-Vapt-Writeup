# F-006: Private Repository Enumeration Gap (Resolved)

**SEVERITY: INFO**

Initially logged as an open mismatch: 4 Jenkins jobs had no corresponding repo visible via Gitea's anonymous `/explore/repos` listing. This was fully explained once leaked credentials ([F-010](../F-010-jenkins-secrets-exposure-console-logs.md)) were used to query Gitea's API directly as the `jenkins` service account: all four repos exist and are simply **private**, visible only to specific accounts.

No vulnerability here — correctly resolved as expected access-control behavior, not a bug. Included to illustrate a real investigative arc: an observation that looked like a gap was later explained by a different finding, rather than chased as its own dead-end.
