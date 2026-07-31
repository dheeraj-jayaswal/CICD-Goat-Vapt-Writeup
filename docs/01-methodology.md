# Methodology & Operational Notes

Testing followed four phases, adapted from standard bug-bounty/VAPT methodology for a CI/CD-focused target.

- **Phase 0 — Rules of Engagement & Scope Confirmation.** See [`00-engagement-overview.md`](./00-engagement-overview.md).
- **Phase 1 — Active Fingerprinting.** Service/version identification across all exposed ports. See [`recon/01-fingerprinting.md`](../recon/01-fingerprinting.md).
- **Phase 2 — Vulnerability Identification.** Authenticated enumeration, secrets discovery, pipeline/source analysis. See [`recon/02-authenticated-enumeration.md`](../recon/02-authenticated-enumeration.md).
- **Phase 3 — Exploitation.** Live PoCs, credential validation, impact confirmation. See individual files under [`findings/`](../findings).

## Operational Lessons (documented deliberately — interviews probe process, not just findings)

### Multibranch Pipeline Navigation Pattern
Jenkins Multibranch Pipeline projects and Folder-type jobs are **containers, not directly buildable jobs**. Any attempt to read `lastBuild`, `config.xml`, or trigger `/build` directly against the parent job returns a 302 redirect rather than a meaningful error — easy to misread as a failure. The correct pattern: enumerate the parent's own `jobs[]` array first (lists actual branch/PR sub-jobs), then operate against the sub-job path (`/job/<parent>/job/<branch-or-PR>/...`).

### Session State for Jenkins CSRF Crumbs
Jenkins' crumb issuer ties a crumb value to the HTTP session it was issued in by default. Fetching a crumb and using it in a separate `curl` invocation without a shared cookie jar (`-c`/`-b`) results in a 403 "No valid crumb was included in the request" even though the crumb value itself is correct.

```bash
COOKIES=/tmp/jenkins_cookies.txt
CRUMB=$(curl -s -c $COOKIES -u alice:alice ".../crumbIssuer/api/json" | jq -r '.crumb')
curl -s -b $COOKIES -u alice:alice -X POST -H "Jenkins-Crumb: $CRUMB" ".../build"
```

### Working Directory Discipline
Several `curl -o` (output-to-file) commands appeared to fail silently mid-engagement. Root cause each time was an incorrect relative path caused by not being in the expected shell working directory — `curl` cannot create missing parent directories and aborts before completing the request when the output path is invalid, which can misleadingly present as a network-level failure (HTTP code `000`). **Lesson:** always `cd` to a known absolute path and confirm with `pwd` before any batch of file-writing commands.

### Distinguishing Infrastructure Health Issues from Application-Level Findings
Jenkins became briefly unresponsive (all endpoints returned "Empty reply from server") partway through testing. Rather than assume this was itself a finding (e.g., a DoS caused by build load), container health was checked directly (`docker stats`, `docker logs`, `docker ps`) before concluding anything — confirming the container was healthy and the issue was transient, not a resource-exhaustion vulnerability. **Verifying infrastructure state before attributing a symptom to a security finding is a frequently tested interview competency.**

### CSRF/Nonce Extraction — Anchor on a Stable Attribute, Not Position
Both GitLab's authenticity token and CTFd's login nonce initially failed extraction because the regex assumed two HTML attributes were adjacent (`name="nonce" value="..."`) when the actual markup placed another attribute (`type="hidden"`) between them. **Lesson:** always extract based on a stable anchor (e.g. `id="nonce"`) with a non-greedy match, rather than assuming fixed attribute ordering/adjacency in markup you haven't directly inspected.

See individual finding files for technique-specific discovery notes.
