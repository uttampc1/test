## README.md

```markdown
# FileGuard

**Git-like file integrity monitoring for Linux systems.**

FileGuard tracks filesystem changes against known-good baselines, detects unauthorized modifications, and provides actionable investigation guidance with OS-specific commands.

```
```
$ fileguard diff

FileGuard — Diff
════════════════

  Comparing against: 8009872f...  (post-install)

  2 change(s) detected:

    Modified:  1
    Added:     1

  Severity: 1 critical, 1 warning

  CHANGE TYPE   SEVERITY   PATH                        DETAIL
  ──────────────────────────────────────────────────────────────
  modified      critical!  /etc/shadow                 hash: a4b862→d3b073
  added         warning?   /usr/local/bin/newutil       file 45KB  SETUID

  Scanned 251,319 entries in 8.2s
```

## Why FileGuard?

Existing tools (AIDE, Tripwire, OSSEC) were designed in the early 2000s. They produce flat text diffs, require complex configuration, and offer no guidance on what to do when something changes.

FileGuard is different:

