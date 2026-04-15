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

| Feature | FileGuard | AIDE/Tripwire | Wazuh |
|---------|-----------|--------------|-------|
| **Setup** | `init` → `baseline create` → `diff` | Edit 200-line config, init DB, learn syntax | Deploy server + agent + Elasticsearch |
| **Scan speed** | ~9 seconds (metadata-first, hash only changed files) | 5+ minutes (hashes everything) | Varies |
| **Output** | Actionable with investigation commands | Raw text diff | JSON events (requires Elastic) |
| **Investigation guides** | OS-aware commands, timelines, what to look for | None | None |
| **Infrastructure** | Single binary + SQLite | Single binary + flat DB | Server + Elastic + Kibana |
| **Deduplication** | Content-addressable (baselines share unchanged items) | None (full copy each time) | N/A |
| **Export** | Self-contained JSON (LLM/SIEM ready) | None | Elastic API |

## Installation

### From Source

```bash
git clone https://github.com/uttampc1/fileguard.git
cd fileguard
go build -o fileguard ./...
sudo mv fileguard /usr/local/bin/
```

### Requirements

- Go 1.21+ (build only)
- Linux (tested on Ubuntu 24.04, should work on any Linux)
- SQLite (embedded, no external dependency)
- Root access recommended for full filesystem scanning

## Quick Start

### 1. Initialize

```bash
sudo fileguard init
```

Creates the data directory (`/var/lib/fileguard/` by default) with a config file and database. The default scope monitors `/etc`, `/bin`, `/sbin`, `/usr`, `/lib`, and `/boot`.

### 2. Create a Baseline

```bash
sudo fileguard baseline create --tag "clean-install"
```

Walks the entire scope, records metadata (size, permissions, ownership, timestamps) and content hashes for every file. This is your known-good reference state.

### 3. Check for Changes

```bash
sudo fileguard diff
```

Compares the current filesystem against the baseline. Only hashes files whose metadata (mtime, size) has changed — making scans fast (~9 seconds for 250K+ entries).

### 4. Get Actionable Guidance

```bash
sudo fileguard summary
```

Shows prioritized changes with investigation commands:

```
🔴 CRITICAL (1):

  /etc/shadow — content modified
    hash: a4b862→d3b073  size: 1,234B→1,256B  owner: root:shadow

    Timeline:
      Known good:    2026-04-02 10:00 (baseline creation)
      File modified: 2026-04-02 15:00 (from mtime)
      Focus window:  5 hours

    Why:   Hashed password storage — all user passwords are stored here.

    Investigate:
      1. Check recent password-related activities
         $ grep -E 'useradd|userdel|passwd' /var/log/auth.log | tail -20
         Look for: useradd (new account), userdel (removed), passwd (password change)

      2. List non-system users and compare with expected
         $ awk -F: '$3>=1000 && $3<65534 {print $1, $3, $7}' /etc/passwd
         Look for: unexpected users with UID >= 1000
```

### 5. Generate a Report

```bash
sudo fileguard report -o integrity-report.txt
sudo fileguard report --format csv -o changes.csv
```

Produces a complete compliance document or CSV for audit/archive.

### 6. Export for Automation

```bash
sudo fileguard export -o scan-data.json
```

Self-contained JSON with complete before/after state for every change. Designed for LLM consumption, SIEM ingestion, or custom automation pipelines.

## Timestamp Forgery Detection

FileGuard detects a sophisticated attack that no competing tool catches: **mtime forgery**.

### The Attack

An attacker modifies a sensitive file (e.g., `/etc/shadow`) and then resets the modification time (`mtime`) to its original value using `touch -t`. Most integrity monitoring tools (AIDE, Tripwire, OSSEC, Wazuh) rely on `mtime` to detect changes — if `mtime` looks unchanged, they skip the file entirely.

```bash
# Attacker modifies /etc/shadow
echo "backdoor:x:0:0::/root:/bin/bash" >> /etc/shadow

# Attacker hides the change by resetting mtime
touch -t 202604011000.00 /etc/shadow
```
## Commands

### Core Workflow

| Command | Purpose |
|---------|---------|
| `fileguard init` | Create data directory, config, and database |
| `fileguard baseline create` | Capture current filesystem state as a reference |
| `fileguard diff` | Compare live filesystem against the baseline |
| `fileguard status` | Quick health check — are there changes? |

### Reporting

| Command | Purpose |
|---------|---------|
| `fileguard summary` | Actionable output with investigation guides |
| `fileguard report` | Formatted compliance report (text/CSV) |
| `fileguard export` | Complete JSON dump for automation/LLM |

### Configuration

| Command | Purpose |
|---------|---------|
| `fileguard scope list` | Show all monitored paths with severity |
| `fileguard scope show <path>` | Detail view of a specific scope |
| `fileguard scope add <path>` | Add a path to monitoring |
| `fileguard scope set <path>` | Change severity or label |
| `fileguard scope remove <path>` | Remove a path from monitoring |

### History

| Command | Purpose |
|---------|---------|
| `fileguard scans` | List past scan runs with statistics |
| `fileguard log` | List all baselines |

---

## Command Reference

### `fileguard init`

```bash
fileguard init [--non-interactive]
fileguard init [--non-interactive]
```

Creates the data directory with `config.yaml` and `fileguard.db`. The `--non-interactive` flag skips prompts and uses defaults.

Default scope:
- `/etc` (warning) — System configuration
- `/bin` (critical) — System binaries
- `/sbin` (critical) — System admin binaries
- `/usr` (warning) — User programs
- `/lib` (warning) — System libraries
- `/boot` (critical) — Boot files

Default exclusions: `/usr/share/doc`, `/usr/share/man`, `/usr/share/info`

### `fileguard baseline create`

```bash
fileguard baseline create [--tag <name>] [-m <message>]
```

Two-pass scan:
1. **Walk** — traverses the filesystem, records metadata (permissions, ownership, timestamps, inode numbers)
2. **Hash** — computes SHA-256 content hashes for regular files

Content-addressable storage: if a file's metadata+hash matches a previous baseline, the existing record is reused. Second and subsequent baselines are nearly instant for unchanged files.

```bash
# First baseline after OS installation
fileguard baseline create --tag "clean-install" -m "Fresh Ubuntu 24.04 install"

# After installing application stack
fileguard baseline create --tag "post-deploy" -m "App deployed, services configured"
```

### `fileguard diff`

```bash
fileguard diff
```

Compares the current filesystem against the HEAD (most recent) baseline. Detects:

| Change Type | Description |
|-------------|-------------|
| `added` | File/directory exists now but not in baseline |
| `modified` | Content hash changed (file was edited) |
| `deleted` | File/directory was in baseline but no longer exists |
| `permission_changed` | File mode changed (e.g., 0644→0755) |
| `ownership_changed` | User or group ownership changed |
| `type_changed` | Entry type changed (e.g., file→symlink) |
| `inode_replaced` | Same path but different inode (file deleted and recreated) |
| `inaccessible` | Parent directory is unreadable (not reported as deleted) |

**Metadata-first optimization:** Only files whose mtime or size changed since the baseline are re-hashed. Unchanged files are verified by metadata alone. This makes scans ~10x faster than tools that hash everything.

**Scope change detection:** If `config.yaml` was modified since the baseline was created (paths added/removed, severity changed), the diff detects and reports scope changes separately from filesystem changes.

### `fileguard status`

```bash
fileguard status [--format text|json] [--quiet]
```

Quick health check without full change details.

```bash
# Human-readable
fileguard status

# Machine-readable
fileguard status --format json

# Exit code only (for scripts/monitoring)
fileguard status --quiet
# Exit 0 = clean, Exit 1 = changes detected
```

### `fileguard summary`

```bash
fileguard summary [--from <baseline>]
```

Concise, prioritized output with actionable investigation guidance. Changes are grouped by severity:

- **🔴 CRITICAL** — Full investigation guide with OS-specific commands, timeline, and what to look for
- **⚠️ WARNING** — Investigation guide with relevant commands
- **ℹ️ INFO** — Grouped by directory, summarized

Each critical/warning change includes:
- **Timeline** — when the file was last known good, when it was modified, when detected
- **Why** — explanation of why this file matters (from sensitive path patterns)
- **Investigation steps** — exact commands to run, adapted for the OS (Ubuntu/Debian uses `dpkg`, RHEL uses `rpm`, etc.)
- **What to look for** — specific patterns in command output that indicate compromise
#### Built-in Investigation Guides

| # | Trigger | Severity | What It Covers |
|---|---------|----------|----------------|
| 1 | `/etc/shadow` modified | Critical | Password hash changes, new accounts, empty passwords |
| 2 | `/etc/passwd` modified | Critical | Account manipulation, UID 0 accounts, shell changes |
| 3 | `/etc/sudoers` changed | Critical | Privilege escalation, permission misconfiguration |
| 4 | `/etc/ssh/sshd_config` modified | Critical | SSH backdoor settings, key authentication changes |
| 5 | SSH host keys changed | Critical | Man-in-the-middle indicators, key rotation |
| 6 | Setuid/setgid binary | Critical | Privilege escalation persistence, package verification |
| 7 | System binary modified | Critical | Rootkit detection, package integrity verification |
| 8 | Cron jobs changed | Warning | Scheduled task persistence, reverse shells |
| 9 | PAM modules modified | Critical | Authentication bypass, credential logging |
| 10 | `/etc/hosts` modified | Warning | DNS hijacking, traffic redirection |

### `fileguard report`

```bash
fileguard report [--format text|csv] [-o <file>] [--baseline <tag>] [--scan <id>]
```

Generates a complete, formatted document for compliance audits or incident documentation.

**Text format** — structured sections: Machine, Baseline, Scan, Summary, Changes (with investigation guides), Footer.

**CSV format** — one row per change with all before/after state columns. Suitable for spreadsheet analysis or import into ticketing systems.

```bash
# Text report to stdout
fileguard report

# Write to file
fileguard report -o /var/reports/integrity-$(date +%Y%m%d).txt

# CSV for spreadsheet analysis
fileguard report --format csv -o changes.csv

# Report against a specific baseline
fileguard report --baseline clean-install
```

### `fileguard export`

```bash
fileguard export [--baseline <tag>] [--scans <n|all>] [--severity <level>] [-o <file>]
```

Complete JSON dump of all scan data. Designed for:
- **LLM consumption** — feed directly to an AI assistant for analysis
- **SIEM ingestion** — structured events with before/after state
- **Custom automation** — parse with `jq`, Python, or any JSON-capable tool
```bash
# Default: HEAD baseline, last 10 scans
fileguard export

# All scans, write to file
fileguard export --scans all -o full-export.json

# Only warning and critical changes
fileguard export --severity warning

# Specific baseline
fileguard export --baseline clean-install -o baseline1-export.json
```

**Export structure:**
```
fileguard_export
├── metadata (version, timestamp, tool version)
├── machine (name, hostname, OS)
├── total_baselines
├── baselines_overview (all baselines with counts)
├── baseline_detail (HEAD or specified)
│   ├── metadata + scope config
│   ├── clean_scan_summary (count + date range)
│   ├── scans_with_changes (full before/after state)
│   └── aggregate_summary (unique paths, distributions)
└── freshness (time since last scan, stale flag)
```

### `fileguard scope`

```bash
# List all scopes
fileguard scope list [--format text|json]

# Detail view
fileguard scope show /etc

# Add a new scope
fileguard scope add /var/log --severity warning --label "System logs"

# Change severity
fileguard scope set /etc --severity critical

# Change label
fileguard scope set /etc --label "Critical system config"

# Remove from monitoring
fileguard scope remove /var/log --force
```

Severity levels:
- **critical** — System binaries, boot files, authentication config. Changes here demand immediate investigation.
- **warning** — Application config, user programs. Changes should be reviewed.
- **info** — Log directories, temporary files. Changes are noted but typically expected.

### `fileguard scans`

```bash
fileguard scans
```
Lists all past scan runs with statistics:

```
FileGuard — Scan History
════════════════════════

  ID         BASELINE       DATE                 DURATION   SCANNED   HASHED  CHANGES
  ─────────────────────────────────────────────────────────────────────────
  a1b2c3d4   post-install   2026-04-09 19:25     8.2s       251,321   3       2
  b2c3d4e5   post-install   2026-04-08 09:00     8.1s       251,319   0       —
  c3d4e5f6   clean-install  2026-04-07 15:30     8.3s       251,319   0       —
```

### `fileguard log`

```bash
fileguard log
```

Lists all baselines with their creation dates and entry counts.

---

## Configuration

### Data Directory

Default: `/var/lib/fileguard/`

Override with `--data-dir`:

```bash
fileguard --data-dir /opt/fileguard diff
```

Or set the environment variable:

```bash
export FILEGUARD_DATA_DIR=/opt/fileguard
fileguard diff
```

### config.yaml

```yaml
machine_name: web-server-01
default_severity: warning

scope:
  paths:
    - path: /etc
      severity: warning
      label: System configuration
    - path: /bin
      severity: critical
      label: System binaries
    - path: /sbin
      severity: critical
      label: System admin binaries
    - path: /usr
      severity: warning
      label: User programs
    - path: /lib
      severity: warning
      label: System libraries
    - path: /boot
      severity: critical
      label: Boot files

  excluded:
    - /usr/share/doc
    - /usr/share/man
    - /usr/share/info
```

### Custom Investigation Guides

Create `<data-dir>/guides.yaml` to add your own investigation guides:

```yaml
guides:
  - id: myapp-config
    path_pattern: "/opt/myapp/config/*"
    change_types: [modified, added, deleted]
    severity: warning
    why: "Application configuration — affects service behavior"
    context: >
      Changes to application configuration files may affect service
      behavior, security settings, or database connections. Verify
      changes were deployed through the authorized release process.
    steps:
      - description: "Check application logs for config reload"
        command: "tail -50 /var/log/myapp/app.log | grep -i config"
        look_for: "Config reload messages around the change time"
      - description: "Validate config syntax"
        command: "/opt/myapp/bin/validate-config"
        look_for: "Syntax errors or warnings"
      - description: "Check deployment history"
        command: "grep myapp /var/log/auth.log | tail -10"
        look_for: "SSH sessions or sudo usage by deployment users"
```
Custom guides override built-in guides when the path pattern matches.

---

## Typical Workflows

### Post-Installation Hardening

```bash
# 1. Fresh OS install — capture clean state
sudo fileguard init
sudo fileguard baseline create --tag "clean-os"

# 2. Install and configure applications
apt install nginx postgresql
systemctl enable nginx postgresql
# ... configure services ...

# 3. Capture hardened state
sudo fileguard baseline create --tag "hardened"

# 4. Schedule regular checks
sudo fileguard diff  # run via cron
```

### Incident Investigation

```bash
# Alert: suspicious activity detected

# 1. Immediate scan
sudo fileguard diff

# 2. Get actionable investigation steps
sudo fileguard summary

# 3. Generate incident report
sudo fileguard report -o /tmp/incident-$(date +%Y%m%d-%H%M).txt

# 4. Export for forensic analysis
sudo fileguard export -o /tmp/forensic-export.json
```

### Compliance Audit

```bash
# Generate compliance report
sudo fileguard report -o audit-report.txt

# CSV for spreadsheet review
sudo fileguard report --format csv -o audit-changes.csv

# Full data export for the auditor's tooling
sudo fileguard export -o audit-export.json
```
### Monitoring with Cron

```bash
# /etc/cron.d/fileguard
# Run integrity check every 6 hours
0 */6 * * * root /usr/local/bin/fileguard diff >> /var/log/fileguard-diff.log 2>&1

# Check exit code for alerting
0 */6 * * * root /usr/local/bin/fileguard status --quiet || echo "INTEGRITY ALERT" | mail -s "FileGuard: changes detected on $(hostname)" security@company.com
```

### Monitoring with systemd Timer

```ini
# /etc/systemd/system/fileguard-check.service
[Unit]
Description=FileGuard integrity check

[Service]
Type=oneshot
ExecStart=/usr/local/bin/fileguard diff
```

```ini
# /etc/systemd/system/fileguard-check.timer
[Unit]
Description=Run FileGuard integrity check every 6 hours

[Timer]
OnCalendar=*-*-* 00/6:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl enable --now fileguard-check.timer
```

---

## Architecture

```
config.yaml ───────────────────┐
                                ▼
                    ┌──────────────────────┐
                    │   fileguard (CLI)     │
                    │                      │
                    │  cmd/                │
                    │  ├── init.go         │
                    │  ├── baseline.go     │
                    │  ├── diff.go         │
                    │  ├── status.go       │
                    │  ├── scope.go        │
                    │  ├── summary.go      │
                    │  ├── report.go       │
                    │  └── export.go       │
                    └──────────┬───────────┘
                               │
            ┌──────────────────┼──────────────────┐
            ▼                  ▼                  ▼
   internal/walker/    internal/diff/     internal/guides/
   └── walk.go         ├── engine.go      ├── guides.go
                       ├── scope.go       ├── builtin.go
                       └── types.go       └── render.go
            │                  │
            └────────┬─────────┘
                     ▼
              internal/store/
              ├── store.go
              └── schema.go
                     │
                     ▼
              fileguard.db (SQLite)
              ├── baselines
              ├── baseline_members
              ├── file_record_items
              ├── scan_runs
              ├── change_events
              ├── sensitive_paths
              └── schema_version (v6)
```
### Storage Model

**Content-addressable deduplication:** Each file's metadata is stored as a `file_record_item` keyed by a hash of its attributes. When creating a new baseline, if a file hasn't changed, the existing item is reused — no new row is written. This means the second baseline (and subsequent ones) only store new/changed items.

**Baseline → Members → Items:**
- `baselines` — metadata (tag, machine, scope config, timestamps)
- `baseline_members` — maps (baseline_id, path) → item_id
- `file_record_items` — content-addressable metadata (hash, size, mode, ownership, timestamps)

**Scan tracking:**
- `scan_runs` — execution metadata (duration, entries scanned/hashed/skipped)
- `change_events` — individual changes with full before/after state JSON

### Performance

| Metric | Value | Notes |
|--------|-------|-------|
| First baseline | ~19s | 250K entries, full hash |
| Subsequent baselines | ~10s | Dedup: only new items stored |
| Diff (no changes) | ~8s | Metadata comparison only, no hashing |
| Diff (few changes) | ~8s | Only re-hashes changed files |
| Database size | ~80MB | 250K entries, single baseline |
| Database growth per baseline | ~0 | Unchanged items are reused |

---

## Roadmap

### Phase 1 (Current) — File Integrity Monitoring ✅

- [x] Baseline creation with content-addressable storage
- [x] Fast metadata-first diff engine
- [x] Inaccessible parent detection (no false "deleted" reports)
- [x] Scope change detection
- [x] CLI scope management (5 subcommands)
- [x] Status command (text/JSON/quiet)
- [x] Export (structured JSON for LLM/SIEM)
- [x] Summary (actionable output with 10 investigation guides)
- [x] Report (text/CSV compliance format)
- [x] 117+ automated tests

### Phase 2 — Deep Analysis & Restore

- [ ] File content storage (content-addressable blobs)
- [ ] `fileguard checkout` — restore any file to baseline state
- [ ] Executable behavior analysis — scan binaries for network ops, IPC, suspicious patterns
- [ ] Live process correlation — map filesystem executables to running processes and open ports
- [ ] Custom investigation guides via CLI
- [ ] HTML report format with expandable sections

### Phase 3 — Enterprise

- [ ] Multi-machine agent/server architecture
- [ ] Fleet management web UI
- [ ] API server
- [ ] Alert integrations (PagerDuty, Slack, email)
- [ ] Compliance report templates (CIS, PCI-DSS, SOC2)
- [ ] Role-based access control

---
## License

[TBD]

---

## Contributing

[TBD]

