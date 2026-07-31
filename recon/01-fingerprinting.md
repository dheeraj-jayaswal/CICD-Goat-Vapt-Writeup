# Phase 1 — Active Fingerprinting

Goal: identify all live services, their exact versions, and initial unauthenticated attack surface across the exposed ports.

## Port and Service Discovery

```bash
# nmap - version detection + default scripts on known ports
nmap -sV -sC -p 3000,4000,8000,8080,8008,2222,50000 -oN nmap_full.txt localhost
```

- `-sV` : version detection (banner grabbing to fingerprint exact software/version)
- `-sC` : default NSE scripts (safe category) — HTTP titles, common misconfig banners
- `-p` : explicit port list — already known from lab documentation, avoids a wasteful full range scan
- `-oN` : normal-format output saved to file for the report/journal

### Result — service map

| Port | Service | Version / Notes |
|---|---|---|
| 2222 | SSH | OpenSSH 9.7 |
| 3000 | HTTP | Gitea 1.16.5 (Go net/http) |
| 4000 | HTTP | GitLab (nginx front-end), stock robots.txt confirmed |
| 8000 | HTTP | CTFd (gunicorn) — "CI/CD Goat" title confirmed |
| 8008 | HTTP | lighttpd 1.4.76 — blanket 403 (production simulation) |
| 8080 | HTTP | Jenkins 2.332.1 / Jetty 9.4.43.v20210629 |
| 50000 | HTTP | Jenkins inbound agent (JNLP) port — flagged for later, not touched directly |

## HTTP Fingerprinting

```bash
# httpx - technology and title fingerprinting across all HTTP services
printf "http://localhost:3000\nhttp://localhost:4000\nhttp://localhost:8000\nhttp://localhost:8080\n" \
  | httpx -title -status-code -tech-detect -server -ip -cl
```

- `-title` : pulls `<title>` tag — quick way to confirm which service is which
- `-status-code` : HTTP response code
- `-tech-detect` : Wappalyzer-style fingerprinting (framework, JS libs, server headers)
- `-server` : Server header value
- `-ip` / `-cl` : resolved IP and content-length

**Key result:** Jenkins returned 403 on root (anonymous UI browsing blocked) while still fingerprinting cleanly via headers — the first hint of an inconsistency between UI-level and API-level authorization, investigated further below.

## Unauthenticated API Probing

**Jenkins — anonymous API access check**

```bash
curl -s http://localhost:8080/api/json | jq .
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080/asynchPeople/
curl -s http://localhost:8080/whoAmI/api/json
```

Result: `/whoAmI/api/json` returned `{"anonymous":true,"authenticated":true,...}` — confirming anonymous sessions carry some baseline authority even though the root UI returns 403. An early signal that different Jenkins endpoints enforce authorization inconsistently.

**Gitea — version and public repo listing**

```bash
curl -s http://localhost:3000/api/v1/version
curl -s http://localhost:3000/explore/repos
```

Result: Gitea 1.16.5, with 7 publicly visible repos across two orgs (`Cov/reportcov` and six `Wonderland/*` repos: caterpillar, dormouse, twiddledum, twiddledee, duchess, dodo).

**GitLab — robots.txt review**

```bash
curl -s http://localhost:4000/robots.txt
```

Result: standard GitLab stock robots.txt (verified against upstream `gitlab-org/gitlab` routes.rb) — not a custom disclosure, ruled out as a lead.
