# JFrog Platform Scripts

Standalone shell scripts for day-to-day JFrog Platform administration — storage
audits, cleanup helpers, and other operational utilities for Artifactory. Each
script is self-contained and documented via `--help`.

## Requirements

Common across scripts in this repo (individual scripts may need more):

- bash 3.2+
- `jq` 1.6+ (needs `--rawfile` and regex support)
- `curl`, `awk`, `sed`, `xargs` (with `-P`/parallel support)

Scripts check their own prerequisites on startup and warn if something's missing.

## Scripts

### `jf_archive_summary.sh`

Storage/archive audit for an Artifactory instance — how much space is actually
in use, and how much of it is realistically reclaimable.

A plain AQL query or API call can't answer this reliably: AQL caps out at
500K results per query, silently drops rows from repos your token can't fully
see, and has no concept of checksum-based dedup — so raw artifact counts and
even UI retention-policy dry-runs can overstate or misjudge what's actually
safe to reclaim. This script handles pagination, permission-gap detection,
and cross-repo checksum dedup so the numbers it reports are accurate at any
scale, including instances with 100M+ artifacts.

**What it reports:**
- Deduplicated storage totals, bucketed by last-download age (last year / 1-3y / 3-5y / 5y+ / never)
- Breakdown by package type, and the single largest repo
- A gap check naming any repos your credential may be missing data from

**Usage:**

```bash
chmod +x jf_archive_summary.sh
./jf_archive_summary.sh --dry-run   # preview scope and time estimate, no data fetched
./jf_archive_summary.sh             # full run
./jf_archive_summary.sh --csv       # also export raw per-artifact rows to CSV

You'll be prompted for your instance name/URL and credentials. Use an
admin-level token — a non-admin or narrowly-scoped token can cause AQL to
silently under-report, and the script will warn you if it detects this.

Run ./jf_archive_summary.sh --help for all flags.

Adding a new script

- Keep it self-contained (no shared libs) and support -h/--help
- Check prerequisites up front rather than failing deep into execution
- Track the script's version in an internal variable, not the filename
- Add a section to this README describing what it does and why
```

![jf_archive_summary.sh output](images/jf_archive_summary%20example.jpg)