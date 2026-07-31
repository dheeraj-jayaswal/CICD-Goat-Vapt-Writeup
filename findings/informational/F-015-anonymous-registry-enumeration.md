# F-015: Anonymous Container Registry Enumeration

**SEVERITY: INFO**

Registry repository/tag listing succeeded with zero authentication:

```bash
curl -s "http://localhost:4000/api/v4/projects/4/registry/repositories"
# -> 200 OK, returns repository list with no token supplied
curl -s "http://localhost:4000/api/v4/projects/4/registry/repositories/2/tags"
# -> 200 OK, returns {"name":"latest", ...}
```

Low-severity information disclosure: image naming and tagging conventions are enumerable by anyone, without credentials. Not independently exploitable, but consistent with this environment's broader pattern of looser-than-ideal anonymous access boundaries, and worth flagging in a real engagement as a registry-hardening recommendation (require auth for repository/tag listing even on public projects).
