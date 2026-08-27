#!/bin/bash
set -euo pipefail

SCRIPT_VERSION="1.1.0"
SCRIPT_NAME="${BASH_SOURCE[0]##*/}" # bash parameter expansion, not `basename`, so it isn't a new external prereq

# Anchor all script-generated files to the script's own directory, regardless
# of where it's invoked from. Computed up front since the intro banner, the
# CSV export's default filename, and the disk-space check all need it.
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
OUT_DIR="$SCRIPT_DIR/aql_pages" # created on demand, by the fallback or the evidence check

# Colorize the intro banner and the % OF BINARY STORAGE bars (step 6) only
# when stdout is an actual terminal, so piping/redirecting output to a file
# never embeds raw ANSI escape codes.
USE_COLOR=false
[ -t 1 ] && USE_COLOR=true
if [ "$USE_COLOR" = "true" ]; then
  C_BOLD=$'\033[1m'; C_DIM=$'\033[2m'; C_GREEN=$'\033[92m'; C_YELLOW=$'\033[93m'; C_RED=$'\033[91m'; C_RESET=$'\033[0m'
else
  C_BOLD=""; C_DIM=""; C_GREEN=""; C_YELLOW=""; C_RED=""; C_RESET=""
fi

# The intro banner's block-art logo and its divider line use Unicode
# box-drawing characters, which render as mojibake outside a UTF-8 locale
# (e.g. a bare "C"/"POSIX" locale, common on minimal/CI shells) -- checked
# via the standard locale env var fallback chain rather than invoking
# `locale` itself, to avoid a new external command dependency for a purely
# cosmetic decision. A plain-ASCII banner is used instead when this is false.
UTF8_LOCALE=false
case "${LC_ALL:-${LC_CTYPE:-${LANG:-}}}" in
*UTF-8* | *utf8*) UTF8_LOCALE=true ;;
esac

# --dry-run: connect and size up the instance, print the same pre-flight
# estimate the real fallback would show, then stop before fetching any
# artifact data.
# --csv[=FILE]: in addition to the printed report, dump the raw (non-
# deduplicated) per-artifact rows fetched along the way to a CSV file --
# FILE defaults to archive_export_<instance>_<timestamp>.csv (step 2 below,
# once the instance name is known) if omitted.
DRY_RUN=no
CSV_EXPORT=no
CSV_FILE=""
HELP=no
while [ $# -gt 0 ]; do
  case "$1" in
  --dry-run) DRY_RUN=yes ;;
  --csv) CSV_EXPORT=yes ;;
  --csv=*) CSV_EXPORT=yes; CSV_FILE="${1#--csv=}" ;;
  -h | --help) HELP=yes ;;
  *)
    echo "Usage: $SCRIPT_NAME [--dry-run] [--csv[=FILE]] [-h|--help]" >&2
    exit 1
    ;;
  esac
  shift
done

# Logo rendered with the "ANSI Shadow" figlet font (pyfiglet), matching the
# reference banner's block-shadow style. Only the 5-letter product name is
# rendered as block art -- the full "JFrog Artifactory Archive Summary"
# title would run well past 200 columns at this scale, so the rest of the
# name becomes the plain-text tagline underneath instead, same as the
# reference banner pairs a short block-art name with a plain-text tagline.
print_banner() {
  if [ "$UTF8_LOCALE" = "true" ]; then
    printf '%s' "${C_GREEN}${C_BOLD}"
    cat <<'LOGO'
     ██╗███████╗██████╗  ██████╗  ██████╗
     ██║██╔════╝██╔══██╗██╔═══██╗██╔════╝
     ██║█████╗  ██████╔╝██║   ██║██║  ███╗
██   ██║██╔══╝  ██╔══██╗██║   ██║██║   ██║
╚█████╔╝██║     ██║  ██║╚██████╔╝╚██████╔╝
 ╚════╝ ╚═╝     ╚═╝  ╚═╝ ╚═════╝  ╚═════╝
LOGO
    printf '%s\n' "$C_RESET"
  else
    printf '%s\n' "${C_GREEN}${C_BOLD}== JFROG ==${C_RESET}"
  fi
  printf '%s\n\n' "${C_BOLD}  v${SCRIPT_VERSION}${C_RESET}  |  Artifactory Archive Summary -- storage audit, archive analysis, CSV export"
}

print_usage() {
  printf '%s\n' "${C_BOLD}Usage:${C_RESET}"
  printf '%s\n\n' "  ${SCRIPT_NAME} [--dry-run] [--csv[=FILE]] [-h|--help]"
  printf '%s\n' "${C_BOLD}Parameters:${C_RESET}"
  printf '%s\n' "  ${C_GREEN}--dry-run${C_RESET}       Connect, size up the instance, print the pre-flight time estimate,"
  printf '%s\n' "                  then exit before fetching any artifact data."
  printf '%s\n' "  ${C_GREEN}--csv[=FILE]${C_RESET}    Also export the raw, non-deduplicated per-artifact data (artifact"
  printf '%s\n' "                  name, package type, binary size, date last downloaded, sha256) to a"
  printf '%s\n' "                  CSV file. FILE defaults to archive_export_<instance>_<timestamp>.csv"
  printf '%s\n' "                  in this script's own directory. Has no effect with --dry-run, since"
  printf '%s\n' "                  no artifact data is fetched in that mode."
  printf '%s\n\n' "  ${C_GREEN}-h, --help${C_RESET}      Show this help and exit."
  printf '%s\n' "You'll also be prompted interactively for the Artifactory instance and credentials --"
  printf '%s\n' "previously used ones, if any, are offered as a pick-list."
}

print_banner
if [ "$HELP" = "yes" ]; then
  print_usage
  exit 0
fi

if [ "$DRY_RUN" = "yes" ]; then
  printf '%s\n' "${C_DIM}Mode: dry run -- no artifact data will be fetched.${C_RESET}"
  [ "$CSV_EXPORT" = "yes" ] && printf '%s\n' "${C_DIM}Note: --csv has no effect in --dry-run mode.${C_RESET}"
  echo
fi
BIG_LIMIT=50000 # per-repo page size, within the fallback
# The single unscoped, all-repos attempt's limit -- and, via ARTIFACTS_COUNT,
# the threshold for even trying it. 500,000 is AQL's own documented ceiling
# (aql.search.query.max.limit); a query at that size against ~190,000 real
# results took about 6 seconds and 60MB. Any instance under this size
# finishes in one call, skipping the per-repo fallback (and all of its
# repo-enumeration edge cases) entirely.
FASTPATH_LIMIT=500000
# Concurrency for the per-repo fallback, the evidence check, and the page-
# projection step. This is network-latency-bound, not CPU-bound (each worker
# mostly waits on Artifactory's response), so it scales well past the CPU
# core count -- on a ~1,000-repo instance, raising this from 8 to 24 cut the
# fallback phase's wall-clock roughly in half, with no rate-limiting either
# way. 16 is a conservative middle ground; raise it further on SaaS
# instances, lower it on smaller/self-hosted ones if requests start erroring.
PARALLEL_JOBS=16
# Rough average row size for the --csv export's five columns (artifact name,
# package type, size, last-downloaded date, sha256); not a measurement, just
# a planning figure shared by the disk-space check below and the CSV
# size-estimate confirmation (step 3). 200 is calibrated off a real export
# (~210 bytes/row actual, across 190K rows); actual bytes/row still varies
# by org with how deep/long repo paths and artifact names tend to run.
CSV_ROW_BYTES_ESTIMATE=200

TMP_FILES=() # tracked for cleanup
cleanup() {
  if [ "${#TMP_FILES[@]}" -gt 0 ]; then
    rm -f "${TMP_FILES[@]}" 2>/dev/null || true
  fi
  rm -rf "$OUT_DIR" 2>/dev/null || true
}
trap cleanup EXIT

# Shared by every "this looks risky, proceed anyway?" checkpoint below --
# default is always no, so an unattended/piped run fails closed.
confirm_or_exit() {
  read -rp "${1:-Continue anyway? [y/N]: }" reply
  [[ "$reply" =~ ^[Yy] ]] || exit 1
}

# Status-line helpers: a bullet marks each action, an indented connector
# marks its outcome -- same visual language as Claude Code's own CLI
# narration. Plain-ASCII fallbacks outside a UTF-8 locale, same as the intro
# banner. C_BOLD/C_YELLOW/C_RESET and *_MARK are plain vars, not functions,
# so they're safe to `export` (not `export -f`) into the per-repo fallback's
# parallel xargs workers below, where these functions themselves don't run.
BULLET="●"; CONNECTOR="⎿"; WARN_MARK="▲"; ERROR_MARK="✗"
if [ "$UTF8_LOCALE" = "false" ]; then
  BULLET="*"; CONNECTOR="L"; WARN_MARK="!"; ERROR_MARK="x"
fi

step() { printf '%s\n' "${C_BOLD}${C_GREEN}${BULLET}${C_RESET} $1"; }
step_done() { printf '%s\n' "  ${C_DIM}${CONNECTOR}${C_RESET}  $1"; }
warn() { printf '%s\n' "${C_BOLD}${C_YELLOW}${WARN_MARK}${C_RESET} ${C_YELLOW}$1${C_RESET}" >&2; }
warn_line() { printf '%s\n' "  $1" >&2; }
error() { printf '%s\n' "${C_BOLD}${C_RED}${ERROR_MARK}${C_RESET} ${C_RED}$1${C_RESET}" >&2; }

# --- 0. Verify prerequisites ---
# A missing or incompatible tool otherwise fails much later and confusingly
# (e.g. a jq syntax error deep in step 7) instead of clearly, up front. Each
# check is a minimal functional probe rather than a version-string parse --
# version strings vary by vendor/build (e.g. "jq-1.7.1-apple"), and it's the
# actual missing feature that matters, not the number attached to it.
PREREQ_ISSUES=()

if [ "${BASH_VERSINFO[0]}" -lt 3 ] || { [ "${BASH_VERSINFO[0]}" -eq 3 ] && [ "${BASH_VERSINFO[1]}" -lt 2 ]; }; then
  PREREQ_ISSUES+=("bash $BASH_VERSION is older than the oldest version this script is tested against (3.2)")
fi

for cmd in jq curl awk sed xargs comm mktemp date; do
  command -v "$cmd" >/dev/null 2>&1 || PREREQ_ISSUES+=("'$cmd' is required but not found on PATH")
done

if command -v jq >/dev/null 2>&1; then
  # --rawfile (used to load AQL's per-repo ground truth in step 3) was added
  # in jq 1.6; /dev/null stands in for a real file, since only the flag
  # itself is being probed.
  jq -n --rawfile x /dev/null '$x' >/dev/null 2>&1 \
    || PREREQ_ISSUES+=("jq does not support --rawfile (need jq 1.6 or newer)")
  # sub()'s regex support (used to parse AQL's timestamps in step 6) needs
  # an Oniguruma-enabled jq build. Most distro builds have this, but it's a
  # compile-time option, not guaranteed by version number alone.
  echo '"x"' | jq 'sub("x";"y")' >/dev/null 2>&1 \
    || PREREQ_ISSUES+=("jq was built without regex support (sub()/test() do not work)")
fi

if command -v xargs >/dev/null 2>&1; then
  # -P (parallel execution) is a GNU/BSD extension the per-repo fallback,
  # the evidence check, and the page-projection step all rely on -- not
  # guaranteed by every xargs implementation (e.g. some minimal/busybox
  # builds lack it).
  printf '1\n' | xargs -P 1 -I {} true >/dev/null 2>&1 \
    || PREREQ_ISSUES+=("xargs does not support -P (parallel execution)")
fi

if [ "${#PREREQ_ISSUES[@]}" -gt 0 ]; then
  warn "Some prerequisites were not met:"
  for issue in "${PREREQ_ISSUES[@]}"; do
    warn_line "- $issue"
  done
  warn_line "The script may fail, hang, or produce incomplete results without them."
  echo >&2
  confirm_or_exit
  echo
fi

# --- 1. Artifactory instance ---
# Accepts either a bare instance name ("example") or a full URL
# ("https://example.jfrog.io", with or without a trailing path) and
# normalizes it to the same form either way.
#
# True tab-completion isn't practical for a plain `read` prompt, so instead:
# offer previously-used instances (one saved config file per instance, in
# this directory) as a numbered pick-list -- pick a number, or just type a
# new name/URL as before.
KNOWN_INSTANCES=()
for f in "$SCRIPT_DIR"/.jfrog_aql_config_*; do
  [ -f "$f" ] || continue
  KNOWN_INSTANCES+=("${f##*/.jfrog_aql_config_}")
done

if [ "${#KNOWN_INSTANCES[@]}" -gt 0 ]; then
  printf '%s\n' "${C_BOLD}Previously used instances:${C_RESET}"
  for i in "${!KNOWN_INSTANCES[@]}"; do
    echo "  $((i + 1))) ${KNOWN_INSTANCES[$i]}"
  done
  read -rp "Enter a number above, or a new instance name/URL: " ARTIFACTORY_INPUT
  if [[ "$ARTIFACTORY_INPUT" =~ ^[0-9]+$ ]] && [ "$ARTIFACTORY_INPUT" -ge 1 ] && [ "$ARTIFACTORY_INPUT" -le "${#KNOWN_INSTANCES[@]}" ]; then
    ARTIFACTORY_INPUT="${KNOWN_INSTANCES[$((ARTIFACTORY_INPUT - 1))]}"
  fi
else
  read -rp "Enter your Artifactory instance name or URL (e.g. 'example' or 'https://example.jfrog.io'): " ARTIFACTORY_INPUT
fi
ARTIFACTORY_INSTANCE="${ARTIFACTORY_INPUT#http://}"
ARTIFACTORY_INSTANCE="${ARTIFACTORY_INSTANCE#https://}"
ARTIFACTORY_INSTANCE="${ARTIFACTORY_INSTANCE%%/*}"
ARTIFACTORY_INSTANCE="${ARTIFACTORY_INSTANCE%.jfrog.io}"
ARTIFACTORY_URL="https://${ARTIFACTORY_INSTANCE}.jfrog.io/artifactory"

if [ "$CSV_EXPORT" = "yes" ] && [ -z "$CSV_FILE" ]; then
  CSV_FILE="$SCRIPT_DIR/archive_export_${ARTIFACTORY_INSTANCE}_$(date +%Y%m%d_%H%M%S).csv"
fi

# --- 2. Credentials (offer saved ones if present) ---
# Scoped to this instance, so a saved login for one instance is never
# offered -- silently and misleadingly -- when you're pointed at another.
CONFIG_FILE="$SCRIPT_DIR/.jfrog_aql_config_${ARTIFACTORY_INSTANCE}"
if [ -f "$CONFIG_FILE" ]; then
  # shellcheck disable=SC1090
  source "$CONFIG_FILE"
fi

USE_SAVED="n"
if [ -n "${SAVED_AUTH_USER:-}" ]; then
  read -rp "Saved credentials found for user '$SAVED_AUTH_USER'. Use them? [Y/n]: " USE_SAVED
  USE_SAVED=${USE_SAVED:-Y}
fi

if [[ "$USE_SAVED" =~ ^[Yy] ]] && [ -n "${SAVED_AUTH_USER:-}" ]; then
  AUTH_USER="$SAVED_AUTH_USER"
  AUTH_PASS="$SAVED_AUTH_PASS"
else
  # Same pick-list idea as the instance prompt, for usernames seen across
  # ANY saved config file (you likely use the same one everywhere). Only the
  # username line is read from other instances' files -- never sourced, so
  # their saved passwords/tokens never load into this process.
  KNOWN_USERS=()
  while IFS= read -r u; do
    KNOWN_USERS+=("$u")
  done < <(sed -n 's/^SAVED_AUTH_USER="\(.*\)"$/\1/p' "$SCRIPT_DIR"/.jfrog_aql_config_* 2>/dev/null | sort -u)

  if [ "${#KNOWN_USERS[@]}" -gt 0 ]; then
    printf '%s\n' "${C_BOLD}Previously used usernames:${C_RESET}"
    for i in "${!KNOWN_USERS[@]}"; do
      echo "  $((i + 1))) ${KNOWN_USERS[$i]}"
    done
    read -rp "Enter a number above, or a new username: " AUTH_USER
    if [[ "$AUTH_USER" =~ ^[0-9]+$ ]] && [ "$AUTH_USER" -ge 1 ] && [ "$AUTH_USER" -le "${#KNOWN_USERS[@]}" ]; then
      AUTH_USER="${KNOWN_USERS[$((AUTH_USER - 1))]}"
    fi
  else
    read -rp "Artifactory username: " AUTH_USER
  fi
  read -rsp "Artifactory password/token: " AUTH_PASS
  echo
  read -rp "Save these credentials for next time? [y/N]: " SAVE_CREDS
  if [[ "$SAVE_CREDS" =~ ^[Yy] ]]; then
    cat >"$CONFIG_FILE" <<EOF
SAVED_AUTH_USER="$AUTH_USER"
SAVED_AUTH_PASS="$AUTH_PASS"
EOF
    chmod 600 "$CONFIG_FILE"
    printf '%s\n' "${C_DIM}Credentials saved to $CONFIG_FILE (readable only by you)${C_RESET}"
  fi
fi

echo
step "Connected to ${ARTIFACTORY_INSTANCE}.jfrog.io"
echo

# --- 2b. Verify admin-level access ---
# AQL applies permission filtering to results AFTER .limit(), silently
# dropping rows from repos the token can't fully see rather than erroring --
# so a non-admin or narrowly-scoped token (e.g. one generated by `jf setup`
# for a package-manager client) produces a plausible-looking but silently
# incomplete report instead of a visible failure. /api/security/users/<name>
# is itself admin-gated, so a successful call with "admin": true confirms
# both that the account is an admin AND that this specific token/credential
# carries that admin-level access, not just that the underlying account does.
#
# NOTE: this only confirms PLATFORM admin. It does NOT guarantee visibility
# into every repo -- on instances using JFrog Projects, a repo assigned to a
# project can still be invisible to AQL for a platform admin who isn't a
# member of that project. There's no cheap way to check that here (it would
# mean one extra API call per repo); the discrepancy check at the end of the
# report is the backstop for that case.
step "Verifying admin access"
ADMIN_CHECK_BODY=$(curl -sS -u "$AUTH_USER:$AUTH_PASS" "$ARTIFACTORY_URL/api/security/users/$AUTH_USER")
if ! echo "$ADMIN_CHECK_BODY" | jq -e '.admin == true' >/dev/null 2>&1; then
  warn "Could not confirm '$AUTH_USER' has admin-level access on $ARTIFACTORY_URL"
  warn_line "This usually means the account isn't an admin, or the token/credential in use"
  warn_line "is scoped down even though the underlying account is. Either way, results from"
  warn_line "repos this token can't fully see will be silently dropped, not reported as an"
  warn_line "error, so counts and sizes below may be incomplete."
  echo >&2
  confirm_or_exit "Continue anyway with possibly incomplete results? [y/N]: "
  echo
else
  step_done "done"
fi

# No download-date filter in the query -- every file is needed, not just
# the stale-looking ones, because determining whether a physical binary is
# actually reclaimable requires knowing if it's ALSO referenced by a
# recently-downloaded copy elsewhere in the repo.
#
# Requires a token/user with read access across every repo being scanned:
# Artifactory applies permission filtering to AQL results after the
# .limit() is applied, so a token missing access to some repos can silently
# under-report rather than error.
AQL_CRITERIA='"type":"file"'
INCLUDE_FIELDS='"repo","path","name","size","sha256","created","stat.downloaded"'

run_aql() {
  # $1 = AQL query string, $2 = output file
  curl -sS -X POST -u "$AUTH_USER:$AUTH_PASS" \
    -H "Content-Type: text/plain" \
    -d "$1" \
    "$ARTIFACTORY_URL/api/search/aql" >"$2"
}

has_results() {
  # $1 = path to a response JSON file. Echoes "yes" if it's a genuine AQL
  # response (valid JSON with a .results array), "no" for anything else --
  # an HTTP error body, an AQL-rejection message, or an empty/partial file
  # left behind by a connection failure.
  jq -e 'has("results") and (.results | type == "array")' "$1" >/dev/null 2>&1 && echo "yes" || echo "no"
}

page_count() {
  # $1 = path to a response JSON file (assumed to already pass has_results).
  # Echoes how many items it contains -- a page with fewer than BIG_LIMIT
  # is necessarily the last one; a full page means there may be more.
  jq '.results | length' "$1"
}

parse_size_to_bytes() {
  # Converts strings like "349.80 GB" or "6.04 TB" to raw bytes (binary
  # units). Awk's string-to-number coercion parses the leading numeric
  # prefix on its own, so the unit is isolated by just stripping digits,
  # dots, and spaces from an uppercased copy of the same string.
  awk -v s="$1" 'BEGIN{
    n = s + 0
    u = toupper(s); gsub(/[0-9. ]/, "", u)
    mult["B"]=1; mult["KB"]=1024; mult["MB"]=1024^2; mult["GB"]=1024^3; mult["TB"]=1024^4; mult["PB"]=1024^5
    printf "%.0f\n", n * (mult[u]+0)
  }'
}

format_count() {
  # Abbreviates a raw item count to the nearest whole M/K, e.g. 35234567 ->
  # "35M", 1234 -> "1K", 500 -> "500". Always an integer, never a decimal.
  local n="$1"
  awk -v n="$n" 'BEGIN{
    if (n >= 1000000) printf "%dM\n", int(n/1000000 + 0.5)
    else if (n >= 1000) printf "%dK\n", int(n/1000 + 0.5)
    else printf "%d\n", n
  }'
}

human_bytes() {
  # Renders a raw byte count as e.g. "38.2 MB" -- used for the CSV export's
  # actual (not estimated) size, once it's known.
  local b="${1:-0}"
  awk -v b="$b" 'BEGIN{
    split("B KB MB GB TB", u, " ")
    i = 1
    while (b >= 1024 && i < 5) { b /= 1024; i++ }
    printf "%.1f %s", b, u[i]
  }'
}

# --- 3. Fetch current total storage size (sanity-check reference only) ---
# This is also the script's first real API call, so it doubles as the
# connectivity/auth check for everything that follows -- a wrong instance
# name or bad credentials fails here, clearly, instead of as a confusing
# jq parse error several steps later.
step "Fetching storage info"
STORAGE_INFO=$(curl -sS -u "$AUTH_USER:$AUTH_PASS" "$ARTIFACTORY_URL/api/storageinfo")
if ! echo "$STORAGE_INFO" | jq -e . >/dev/null 2>&1; then
  error "$ARTIFACTORY_URL/api/storageinfo did not return valid JSON"
  warn_line "Double-check the instance name/URL and credentials. Response was:"
  echo "$STORAGE_INFO" | head -c 500 >&2
  echo >&2
  exit 1
fi
BINARIES_SIZE_HUMAN=$(echo "$STORAGE_INFO" | jq -r '.binariesSummary.binariesSize // "0 B"')
BINARIES_BYTES=$(parse_size_to_bytes "$BINARIES_SIZE_HUMAN")
# Used below to skip the doomed all-repos attempt outright on instances that
# are already known to be far bigger than one page can hold.
ARTIFACTS_COUNT=$(echo "$STORAGE_INFO" | jq -r '.binariesSummary.artifactsCount // "0"' | tr -d ',')
step_done "done"

# --csv size estimate, using this same instance-wide count -- the CSV's rows
# are per-item, not deduplicated by checksum, so ARTIFACTS_COUNT (not the
# dedup totals computed later in step 6) is the right basis. Declining here
# only drops the CSV export; the storage audit itself still runs, since a
# large CSV isn't a reason to abandon the rest of the report. Skipped for
# --dry-run, since --csv already has no effect there.
CSV_SIZE_WARN_BYTES=$((100 * 1024 * 1024)) # 100 MB
if [ "$CSV_EXPORT" = "yes" ] && [ "$DRY_RUN" = "no" ]; then
  CSV_EST_BYTES=$((ARTIFACTS_COUNT * CSV_ROW_BYTES_ESTIMATE))
  if [ "$CSV_EST_BYTES" -gt "$CSV_SIZE_WARN_BYTES" ] 2>/dev/null; then
    warn "--csv is estimated at ~$((CSV_EST_BYTES / 1048576))MB across ~$(format_count "$ARTIFACTS_COUNT") rows"
    warn_line "(a rough estimate from this instance's total artifact count, not a measurement)."
    echo >&2
    read -rp "Continue with CSV export anyway? [y/N]: " CSV_SIZE_REPLY
    if [[ "$CSV_SIZE_REPLY" =~ ^[Yy] ]]; then
      : # proceed with CSV export as requested
    else
      printf '%s\n' "${C_DIM}Skipping CSV export; continuing with the storage audit.${C_RESET}" >&2
      CSV_EXPORT=no
      CSV_FILE=""
    fi
    echo
  fi
fi

# Per-repo ground truth (filesCount / usedSpaceInBytes / projectKey), straight
# from this same storageinfo response -- computed server-side, independent of
# what this credential's own AQL queries can see. Used at the end of the
# report to pinpoint exactly which repos this credential came back short on,
# instead of only a single instance-wide percentage. "TOTAL" is a synthetic
# roll-up row, not a real repo, so it's excluded; older Artifactory versions
# without this field just produce an empty file, which the gap check below
# treats as "nothing to compare" rather than an error.
REPO_GROUNDTRUTH_FILE=$(mktemp)
TMP_FILES+=("$REPO_GROUNDTRUTH_FILE")
echo "$STORAGE_INFO" | jq -r '
  .repositoriesSummaryList[]?
  | select(.repoKey != "TOTAL")
  | [.repoKey, .repoType, (.filesCount // 0), (.usedSpaceInBytes // 0), (.projectKey // "-")]
  | @tsv
' >"$REPO_GROUNDTRUTH_FILE"

# The single biggest repo instance-wide, by artifact count -- computed below
# (step 4), once REPO_TYPES_FILE can tell a real package repo apart from an
# internal one.

# Repository count, from the same storageinfo response -- deliberately the
# raw repositoriesSummaryList length (including its synthetic "TOTAL" row),
# not /api/repositories' length, to match the count shown on the platform
# UI itself, which also includes that row. /api/repositories alone would
# also miss REMOTE repos' separately-counted "-cache" storage and the
# hidden system repos (build-info, platform logs, trash can).
REPO_COUNT=$(echo "$STORAGE_INFO" | jq '(.repositoriesSummaryList // []) | length')

# Disk-space check, now that the instance's real size is known. Every
# fetched AQL page (raw JSON) and its projected TSV land in $SCRIPT_DIR
# until cleanup, so a 100M+-artifact instance can plausibly need tens of
# GB of scratch space. ~250 bytes/item for the raw AQL JSON (repo, path,
# name, size, sha256, created, download stat) plus ~100 bytes/item for its
# projected TSV is a rough per-item estimate, not a measurement; a 2x
# safety margin covers the roughness. Skipped silently if `df` isn't
# available or its output can't be parsed -- this is a helpful heads-up,
# not something worth blocking a run over on its own. Skipped for --dry-run,
# since nothing will actually be fetched or written.
if [ "$DRY_RUN" = "no" ] && command -v df >/dev/null 2>&1; then
  PER_ITEM_BYTES=350
  # --csv adds its own file in the same directory, folded into the same 2x
  # safety margin.
  [ "$CSV_EXPORT" = "yes" ] && PER_ITEM_BYTES=$((PER_ITEM_BYTES + CSV_ROW_BYTES_ESTIMATE))
  REQUIRED_KB=$((ARTIFACTS_COUNT * PER_ITEM_BYTES * 2 / 1024))
  AVAILABLE_KB=$(df -Pk "$SCRIPT_DIR" 2>/dev/null | awk 'NR==2{print $4}')
  if [ -n "$AVAILABLE_KB" ] && [ "$REQUIRED_KB" -gt 0 ] 2>/dev/null && [ "$AVAILABLE_KB" -lt "$REQUIRED_KB" ] 2>/dev/null; then
    warn "An estimated ~$((REQUIRED_KB / 1048576))GB of scratch space may be needed in"
    warn_line "$SCRIPT_DIR (this instance reports $ARTIFACTS_COUNT artifacts), but only"
    warn_line "~$((AVAILABLE_KB / 1048576))GB is free there. This is a rough estimate, not a hard limit -- the"
    warn_line "actual usage depends on how much of that data this credential can see."
    echo >&2
    confirm_or_exit
    echo
  fi
fi

# --- 4. Discover repositories (for the per-package-type breakdown, and
# reused below by the per-repo AQL fallback so it isn't fetched twice) ---
step "Fetching repository list"
REPOS_JSON=$(curl -sS -u "$AUTH_USER:$AUTH_PASS" "$ARTIFACTORY_URL/api/repositories")
step_done "done"
[ "$DRY_RUN" = "yes" ] && echo
# repo -> packageType, as a TSV lookup file for the awk aggregator (step 6)
# rather than a JSON blob, since awk has no JSON parser. A REMOTE repo's
# cached binaries physically live under "<key>-cache" -- an implicit repo
# key that Artifactory never lists as its own /api/repositories entry, so
# without the second line here it maps to nothing and shows up as "Unknown".
REPO_TYPES_FILE=$(mktemp)
TMP_FILES+=("$REPO_TYPES_FILE")
echo "$REPOS_JSON" | jq -r '
  .[]
  | [.key, .packageType],
    (if .type == "REMOTE" then [(.key + "-cache"), .packageType] else empty end)
  | @tsv
' >"$REPO_TYPES_FILE"

# The single biggest repo instance-wide, by artifact count -- from the same
# server-side ground truth (step 3), so it reflects the true instance size
# even for repos this credential can't otherwise see. Picked here, not in
# step 3, since telling a real package repo apart from an internal one needs
# REPO_TYPES_FILE (just built above). Same fallback categories as the awk
# aggregator's own ptype logic (step 6), kept in sync by hand since this
# side is bash/awk-inline, not the aggregator's own awk pass:
#   - Release bundles -- both the auto-created per-project Build Info repo
#     (invisible to /api/repositories, caught by the "-build-info" fallback
#     below) and a real repo whose packageType /api/repositories itself
#     reports as "ReleaseBundles" -- and JFrog Platform's own internal
#     storage (billing/usage logs, full-sync, release bundles v2) are
#     excluded -- none of these are a real, user-facing package repo, so
#     none should be able to win "biggest repo".
#   - The trash can (repoType "NA", so normally invisible to the
#     LOCAL/CACHE/FEDERATED filter) is included by name instead, since a
#     large trash can IS a meaningful, actionable storage-audit signal.
read -r BIGGEST_REPO_KEY BIGGEST_REPO_COUNT BIGGEST_REPO_TYPE <<<"$(awk -F'\t' -v types_file="$REPO_TYPES_FILE" '
  BEGIN {
    while ((getline line < types_file) > 0) {
      split(line, a, "\t")
      if (a[1] != "") reptype[a[1]] = a[2]
    }
    close(types_file)
    platform["jfrog-billing-logs"] = 1
    platform["jfrog-usage-logs"] = 1
    platform["jfrog-full-sync-info"] = 1
    platform["release-bundles-v2-jfds"] = 1
  }
  {
    repo = $1; type = $2; files = $3 + 0
    if (!(type == "LOCAL" || type == "CACHE" || type == "FEDERATED" || repo == "auto-trashcan")) next

    ptype = reptype[repo]
    if (ptype == "") {
      if (repo == "auto-trashcan") ptype = "Trash"
      else if (repo in platform) ptype = "JFrog Platform"
      else if (repo ~ /-build-info$/) ptype = "Release bundles"
      else ptype = "Unknown"
    }
    if (ptype == "JFrog Platform" || ptype == "Release bundles" || ptype == "ReleaseBundles") next

    if (!seen || files > max) { seen = 1; max = files; key = repo; keytype = ptype }
  }
  END { print (key == "" ? "-" : key), max + 0, (key == "" ? "Unknown" : keytype) }
' "$REPO_GROUNDTRUTH_FILE")"
case "$BIGGEST_REPO_TYPE" in
HuggingFaceML) BIGGEST_REPO_TYPE="Hugging Face ML" ;;
Gitlfs) BIGGEST_REPO_TYPE="Git LFS" ;;
esac

# Single snapshot of "now", used consistently for every item's recency
# bucket regardless of how long the fetch takes on a huge instance.
NOW_EPOCH=$(date +%s)
CUTOFF_1Y=$((NOW_EPOCH - 1 * 365 * 86400))
CUTOFF_3Y=$((NOW_EPOCH - 3 * 365 * 86400))
CUTOFF_5Y=$((NOW_EPOCH - 5 * 365 * 86400))

# --- 5. Attempt ONE query across all repos first (fast path) ---
# Skipped outright on instances already known (from storageinfo, step 3) to
# be well beyond one page -- on a 100M-artifact instance this attempt is
# doomed before it starts, and it's a full unscoped table scan, so there's
# no point spending the time (or the load on Artifactory's DB) on it.
NEEDS_FALLBACK=no
PAGE_FILES=()
if [ "$ARTIFACTS_COUNT" -gt "$FASTPATH_LIMIT" ] 2>/dev/null; then
  NEEDS_FALLBACK=yes
elif [ "$DRY_RUN" = "yes" ]; then
  step_done "$(format_count "$ARTIFACTS_COUNT") artifacts -- expected execution time: a few minutes"
  echo
  printf '%s\n' "${C_DIM}Dry run: stopping here.${C_RESET}"
  exit 0
else
  step "Querying all repos"
  QUERY="items.find({${AQL_CRITERIA}}).include(${INCLUDE_FIELDS}).limit(${FASTPATH_LIMIT})"
  FASTPATH_FILE=$(mktemp)
  TMP_FILES+=("$FASTPATH_FILE")
  run_aql "$QUERY" "$FASTPATH_FILE"

  # Fall back to per-repo queries not just when the single query is full
  # (there may be more beyond this one page), but also when it fails
  # outright -- an AQL error body, a gateway error page, etc. A repo-scoped
  # query is smaller and sometimes succeeds where the all-repos one hit a
  # server-side complexity/size limit; if it's a deeper problem (bad auth,
  # endpoint down), the per-repo queries will fail the same way, but at
  # least each failure is now visible instead of silently treated as
  # "0 results".
  if [ "$(has_results "$FASTPATH_FILE")" = "no" ]; then
    warn "Single query did not return a valid result set. Response was:"
    head -c 500 "$FASTPATH_FILE" >&2
    echo >&2
    NEEDS_FALLBACK=yes
  elif [ "$(page_count "$FASTPATH_FILE")" -ge "$FASTPATH_LIMIT" ]; then
    step_done "Single query returned a full page -- there may be more"
    NEEDS_FALLBACK=yes
  else
    step_done "done"
    PAGE_FILES=("$FASTPATH_FILE")
  fi
  [ "$NEEDS_FALLBACK" = "yes" ] && step "Falling back to per-repo queries (up to $PARALLEL_JOBS at a time)"
fi

if [ "$NEEDS_FALLBACK" = "yes" ]; then
  # --- 5b. Fallback: per-repo queries in parallel, each repo paginated via
  # .offset() until a page comes back short of BIG_LIMIT, instead of
  # capping every repo at one page and losing whatever's beyond it. No
  # .sort() is applied, so heavy concurrent writes to a repo mid-scan could
  # in principle shift a handful of rows across the page boundary; accepted
  # here as a storage-report accuracy trade-off, not a correctness guarantee.
  #
  # A keyset ("seek") pagination scheme -- sort by (created, path, name),
  # filter each page to strictly after the previous page's last row -- was
  # tried instead, to avoid .offset()'s per-page cost at very large offsets.
  # It had to be reverted: AQL does not reliably honor .sort() once combined
  # with the compound filter keyset pagination requires, so a real ~20,000-
  # item repo came back in no coherent order, cycling through repeated
  # blocks of timestamps instead of advancing monotonically -- which would
  # silently skip and duplicate real data. .offset() is slower at extreme
  # single-repo scale, but it's the option that's actually correct. ---
  mkdir -p "$OUT_DIR"
  # REMOTE repos have no physical storage under their own key -- their
  # cached content lives under the implicit "<key>-cache" repo, and AQL
  # matches "repo" on that exact physical key. Querying the REMOTE repo's
  # own key returns zero results, so the cache key replaces it here rather
  # than adding to it.
  #
  # VIRTUAL repos are excluded entirely -- they hold no storage of their
  # own. AQL resolves a query against a virtual repo's key by returning its
  # underlying LOCAL/REMOTE-cache/FEDERATED repos' items, tagged with THEIR
  # real repo key, not the virtual repo's. Those underlying repos are
  # already queried directly by this same loop, so including virtual repos
  # here would double-count every item any virtual repo aggregates.
  #
  # storageinfo's own per-repo ground truth (step 3) also reports repos
  # Artifactory hides from /api/repositories entirely -- the auto-created
  # per-project Build Info repo, and JFrog's own platform-internal logs --
  # which are directly queryable via AQL despite being invisible to the
  # repository list, so they're included here too instead of only being
  # flagged as an unreachable gap in step 7.
  #
  # The trash can (auto-trashcan) is the same story -- hidden from
  # /api/repositories, but (unlike ".evidence", see step 7) fully queryable
  # via AQL. Included by name rather than by type (its repoType is "NA",
  # not one of the three above) since it's the one fixed, well-known system
  # repo of that type worth querying; other "NA" entries in the ground
  # truth are synthetic roll-up rows, not real repos.
  #
  # Repos storageinfo's own ground truth already reports as having zero
  # files are skipped outright -- querying them can only ever come back
  # empty, and on a large instance a meaningful fraction of repos are
  # typically empty placeholders. A repo missing from the ground truth
  # entirely (rather than explicitly reporting 0) is NOT skipped -- only an
  # explicit, authoritative "0" is treated as proof there's nothing to fetch.
  ZERO_FILE_REPOS_FILE=$(mktemp)
  TMP_FILES+=("$ZERO_FILE_REPOS_FILE")
  awk -F'\t' '($3+0==0){print $1}' "$REPO_GROUNDTRUTH_FILE" | sort -u >"$ZERO_FILE_REPOS_FILE"

  {
    echo "$REPOS_JSON" | jq -r '
      .[]
      | select(.type != "VIRTUAL")
      | if .type == "REMOTE" then (.key + "-cache") else .key end
    '
    awk -F'\t' '($2=="LOCAL"||$2=="CACHE"||$2=="FEDERATED"){print $1}' "$REPO_GROUNDTRUTH_FILE"
    awk -F'\t' '($1=="auto-trashcan"){print $1}' "$REPO_GROUNDTRUTH_FILE"
  } | sort -u | comm -23 - "$ZERO_FILE_REPOS_FILE" >"$OUT_DIR/repo_list.txt"

  # --- Pre-flight estimate, before spending any time on the fallback itself.
  # One awk pass (the classic two-file NR==FNR join) computes the total AQL
  # calls needed and flags any repo needing >= DOMINATE_PAGES pages, since a
  # single such repo dominates wall-clock on its own: its pages are fetched
  # sequentially within one worker (query_repo() doesn't parallelize within
  # a repo), so PARALLEL_JOBS never helps that repo specifically -- e.g. a
  # single repo holding 100M artifacts alone needs ~2,000 sequential pages
  # at the default BIG_LIMIT, regardless of how many workers are free. The
  # 0.4s/call figure is an empirically observed rate (8 workers -> 50s, 24
  # workers -> 25s, for ~950 mostly single-page repos on one SaaS instance),
  # not a guarantee -- self-hosted instances or heavier network paths could
  # be slower or faster. ---
  step "Computing pre-flight estimate"
  DOMINATE_PAGES=20 # repos needing this many pages or more (>=1M items at the default BIG_LIMIT) are called out individually
  ASSUMED_SECONDS_PER_CALL="0.4"
  DOMINATING_REPOS_FILE=$(mktemp)
  TMP_FILES+=("$DOMINATING_REPOS_FILE")
  PREFLIGHT_SUMMARY=$(awk -F'\t' -v big_limit="$BIG_LIMIT" -v dominate_pages="$DOMINATE_PAGES" -v domfile="$DOMINATING_REPOS_FILE" '
    NR==FNR { filescount[$1] = $3 + 0; next }
    {
      repo = $1
      files = (repo in filescount) ? filescount[repo] : 0
      pages = int((files + big_limit - 1) / big_limit)
      if (pages < 1) pages = 1
      total_calls += pages
      total_repos++
      if (pages >= dominate_pages) {
        printf "%s\t%d\t%d\n", repo, files, pages >>domfile
        if (pages > max_pages) max_pages = pages
      }
    }
    END { printf "%d %d %d\n", total_calls, total_repos, max_pages+0 }
  ' "$REPO_GROUNDTRUTH_FILE" "$OUT_DIR/repo_list.txt")
  read -r TOTAL_CALLS TOTAL_REPOS_TO_QUERY MAX_DOMINATING_PAGES <<<"$PREFLIGHT_SUMMARY"

  # The real floor is whichever is larger: average parallel throughput, or
  # the single slowest repo's own unparallelizable serial chain.
  read -r ETA_SEC ETA_MIN <<<"$(awk -v c="$TOTAL_CALLS" -v p="$PARALLEL_JOBS" -v m="$MAX_DOMINATING_PAGES" -v s="$ASSUMED_SECONDS_PER_CALL" 'BEGIN{
    parallel = (c/p)*s; solo = m*s
    eta = (solo > parallel) ? solo : parallel
    printf "%.0f %.1f", eta, eta/60
  }')"

  step_done "$TOTAL_REPOS_TO_QUERY repo(s), ~$TOTAL_CALLS AQL call(s), roughly ${ETA_MIN} min at $PARALLEL_JOBS workers"
  CONFIRM_FALLBACK=no
  if [ -s "$DOMINATING_REPOS_FILE" ]; then
    echo
    warn "The following repo(s) need $DOMINATE_PAGES+ sequential pages each -- each one"
    warn_line "dominates the fallback's wall-clock on its own, regardless of concurrency,"
    warn_line "since one repo's own pages can't be parallelized:"
    awk -F'\t' '{printf "    - %s: %s artifacts (~%s pages)\n", $1, $2, $3}' "$DOMINATING_REPOS_FILE" >&2
    CONFIRM_FALLBACK=yes
  elif [ "$ETA_SEC" -ge 60 ] 2>/dev/null; then
    CONFIRM_FALLBACK=yes
  fi

  if [ "$DRY_RUN" = "yes" ]; then
    echo
    printf '%s\n' "${C_DIM}Dry run: stopping here.${C_RESET}"
    exit 0
  fi

  if [ "$CONFIRM_FALLBACK" = "yes" ]; then
    echo
    confirm_or_exit "Proceed with the per-repo fallback now? [y/N]: "
    read -rp "Concurrency to use for the fallback [$PARALLEL_JOBS]: " PARALLEL_JOBS_OVERRIDE
    if [[ "$PARALLEL_JOBS_OVERRIDE" =~ ^[0-9]+$ ]] && [ "$PARALLEL_JOBS_OVERRIDE" -ge 1 ]; then
      PARALLEL_JOBS="$PARALLEL_JOBS_OVERRIDE"
    fi
  fi
  echo

  query_repo() {
    local repo="$1" offset=0 page=0 max_pages=1000 count
    while [ "$page" -lt "$max_pages" ]; do
      local outfile="$OUT_DIR/${repo}__${page}.json"
      local query="items.find({\"repo\":\"$repo\",${AQL_CRITERIA}}).include(${INCLUDE_FIELDS}).offset(${offset}).limit(${BIG_LIMIT})"
      run_aql "$query" "$outfile"

      if [ "$(has_results "$outfile")" = "no" ]; then
        printf '%s\n' "  ${C_BOLD}${C_YELLOW}${WARN_MARK}${C_RESET} ${C_YELLOW}repo '$repo' query failed at offset $offset -- data from this repo may be incomplete${C_RESET}" >&2
        rm -f "$outfile"
        return
      fi

      count=$(page_count "$outfile")
      if [ "$count" -lt "$BIG_LIMIT" ]; then
        return
      fi
      page=$((page + 1))
      offset=$((offset + BIG_LIMIT))
    done
    printf '%s\n' "  ${C_BOLD}${C_YELLOW}${WARN_MARK}${C_RESET} ${C_YELLOW}repo '$repo' hit the pagination safety cap ($max_pages pages) -- data from this repo may be incomplete${C_RESET}" >&2
  }
  export -f run_aql query_repo has_results page_count
  export AUTH_USER AUTH_PASS ARTIFACTORY_URL AQL_CRITERIA INCLUDE_FIELDS BIG_LIMIT OUT_DIR C_BOLD C_YELLOW C_RESET WARN_MARK

  xargs -P "$PARALLEL_JOBS" -I {} bash -c 'query_repo "$@"' _ {} <"$OUT_DIR/repo_list.txt"

  # Excluding failed pages individually here (rather than handing the whole
  # batch to one merge step) matters: one invalid-JSON file in the batch
  # would otherwise be indistinguishable from a genuine empty page.
  for f in "$OUT_DIR"/*.json; do
    if [ "$(has_results "$f")" = "yes" ]; then
      PAGE_FILES+=("$f")
    fi
  done
fi

echo

# --- 6. Aggregate every fetched page, then print the binary-storage summary
# tables. Split in two: an awk pass does the O(n) streaming reduction (bucket
# counts, checksum dedup, per-type totals) so memory scales with the number
# of UNIQUE binaries rather than total artifact count -- at 100M+ artifacts,
# loading everything into one jq array for a `group_by` (an in-memory sort)
# is what breaks first, not the network fetch. jq then renders that handful
# of numbers into the summary tables. ---
PROJECT_PROGRAM='
  def epoch: sub("\\.[0-9]+Z$"; "Z") | fromdateiso8601;
  .results[]
  | [
      (.repo // ""),
      (.path // ""),
      (.name // ""),
      (.size // 0),
      (.sha256 // "-"),
      (if (.stats // []) == [] then "-" else (.stats[0].downloaded | epoch) end),
      (if .created == null then "-" else (.created | epoch) end),
      (if (.stats // []) == [] then "-" else .stats[0].downloaded end)
    ]
  | @tsv
'

AWK_PROGRAM_FILE=$(mktemp)
TMP_FILES+=("$AWK_PROGRAM_FILE")
cat >"$AWK_PROGRAM_FILE" <<'AWKEOF'
BEGIN {
  FS = "\t"
  while ((getline line < REPO_TYPES_FILE) > 0) {
    n = split(line, a, "\t")
    if (n == 2) reptype[a[1]] = a[2]
  }
  close(REPO_TYPES_FILE)

  platform["jfrog-billing-logs"] = 1
  platform["jfrog-usage-logs"] = 1
  platform["jfrog-full-sync-info"] = 1
  platform["release-bundles-v2-jfds"] = 1

  rank["lastyear"] = 0
  rank["b1_3"] = 1
  rank["b3_5"] = 2
  rank["b5plus"] = 3
  rank["never_lastyear"] = 4
  rank["never_older"] = 5
  bname[0] = "lastyear"; bname[1] = "b1_3"; bname[2] = "b3_5"; bname[3] = "b5plus"
  bname[4] = "never_lastyear"; bname[5] = "never_older"

  # Same rename map as the printed report's displayname() (step 6's jq
  # program) -- kept in sync by hand since this side is awk, not jq.
  dispname["HuggingFaceML"] = "Hugging Face ML"
  dispname["Gitlfs"] = "Git LFS"

  if (CSV_FILE != "") {
    print "Artifact Name,Package Type,Binary Size (Bytes),Date Last Downloaded,SHA256" > CSV_FILE
  }
}

{
  repo = $1
  path = $2
  name = $3
  size = $4 + 0
  csum = $5
  downloaded = $6
  created = $7
  downloaded_raw = $8

  # Per-repo raw (non-deduplicated) totals -- physical usage, same basis as
  # storageinfo's per-repo usedSpaceInBytes -- for the gap check in step 7.
  foundcount[repo]++
  foundbytes[repo] += size

  # Never downloaded splits by age instead of download recency -- a
  # just-created item hasn't had a chance to be downloaded yet, which is a
  # very different signal from one that's sat untouched for a long time.
  if (downloaded == "-") {
    if (created >= cutoff1y) bucket = "never_lastyear"
    else bucket = "never_older"
  }
  else if (downloaded >= cutoff1y) bucket = "lastyear"
  else if (downloaded >= cutoff3y) bucket = "b1_3"
  else if (downloaded >= cutoff5y) bucket = "b3_5"
  else bucket = "b5plus"

  rawcount[bucket]++

  # Not a "real" package repo, so it's never in reptype: JFrog's own
  # platform-internal storage (billing/usage logs, full-sync, release
  # bundles v2), the auto-created per-project Build Info repo, and the
  # trash can -- broken out as its own type rather than falling into
  # "Unknown" since it's a directly actionable category for a storage
  # reclaim audit (already-deleted, pending purge).
  ptype = reptype[repo]
  if (ptype == "") {
    if (repo == "auto-trashcan") ptype = "Trash"
    else if (repo in platform) ptype = "JFrog Platform"
    else if (repo ~ /-build-info$/) ptype = "Release bundles"
    else ptype = "Unknown"
  }
  if (ptype == "Unknown") unknownrepo[repo] = 1
  # Raw (non-deduplicated) count per type -- same basis as the main table's
  # ARTIFACT COUNT column, so the two tables' counts are directly comparable.
  typerawcount[ptype]++

  # Raw per-artifact CSV row -- one line per fetched item, deliberately NOT
  # deduplicated by checksum (unlike the summary tables above): a storage
  # export is expected to list every physical copy, not just distinct
  # binaries.
  if (CSV_FILE != "") {
    artifact_name = (path == "." || path == "") ? (repo "/" name) : (repo "/" path "/" name)
    dl_display = (downloaded_raw == "-") ? "Never" : substr(downloaded_raw, 1, 10)
    ptype_display = (ptype in dispname) ? dispname[ptype] : ptype
    print csvfield(artifact_name) "," csvfield(ptype_display) "," size "," csvfield(dl_display) "," csvfield(csum) > CSV_FILE
  }

  # Dedup key: checksum if we have one, else a synthetic per-record key
  # (artifacts with no sha256 -- rare, pre-5.5 unmigrated items -- are
  # treated as their own isolated, non-deduplicated group; conservative,
  # since it may undercount dedup space but never overstates it).
  key = (csum == "-") ? ("nocs-" NR) : csum

  # A binary is credited to the LEAST-stale bucket among all its references
  # (rank 0 = most recent wins) -- that's the one that gates full reclaim,
  # and the package type of whichever reference determines that. First-seen
  # wins on ties, an arbitrary but stable tiebreak.
  r = rank[bucket]
  if (!(key in keyrank) || r < keyrank[key]) {
    keyrank[key] = r
    keysize[key] = size
    keytype[key] = ptype
  }
}

END {
  for (k in keyrank) {
    b = bname[keyrank[k]]
    dedupsize[b] += keysize[k]
    t = keytype[k]
    typetotal[t] += keysize[k]
    seentype[t] = 1
  }

  printf "{"

  printf "\"rawcount\":{"
  for (i = 0; i <= 5; i++) printf "%s\"%s\":%d", (i>0?",":""), bname[i], rawcount[bname[i]]+0
  printf "},"

  printf "\"dedupsize\":{"
  for (i = 0; i <= 5; i++) printf "%s\"%s\":%d", (i>0?",":""), bname[i], dedupsize[bname[i]]+0
  printf "},"

  printf "\"typeTotals\":{"
  first = 1
  for (t in seentype) {
    printf "%s%s:%d", (first?"":","), jsonstr(t), typetotal[t]+0
    first = 0
  }
  printf "},"

  printf "\"typeCounts\":{"
  first = 1
  for (t in seentype) {
    printf "%s%s:%d", (first?"":","), jsonstr(t), typerawcount[t]+0
    first = 0
  }
  printf "},"

  printf "\"unknownRepos\":["
  first = 1
  for (r in unknownrepo) {
    printf "%s%s", (first?"":","), jsonstr(r)
    first = 0
  }
  printf "],"

  printf "\"byRepo\":{"
  first = 1
  for (r in foundcount) {
    printf "%s%s:{\"count\":%d,\"bytes\":%d}", (first?"":","), jsonstr(r), foundcount[r], foundbytes[r]
    first = 0
  }
  printf "}"

  printf "}\n"

  if (CSV_FILE != "") close(CSV_FILE)
}

function jsonstr(s) {
  gsub(/\\/, "\\\\", s)
  gsub(/"/, "\\\"", s)
  return "\"" s "\""
}

function csvfield(s) {
  if (index(s, ",") || index(s, "\"") || index(s, "\n")) {
    gsub(/"/, "\"\"", s)
    return "\"" s "\""
  }
  return s
}
AWKEOF

# rpad/lpad/dashes/commas are shared with the gap-report's own jq program
# in step 7; defined once here since both consumers need identical column
# formatting.
JQ_PAD_DEFS='
def rpad(w): (w - (.|length)) as $n | . + (if $n > 0 then ([range(0;$n)] | map(" ") | join("")) else "" end);
def lpad(w): (w - (.|length)) as $n | (if $n > 0 then ([range(0;$n)] | map(" ") | join("")) else "" end) + .;
def dashes(w): [range(0;w)] | map("-") | join("");
def commas:
  . as $s
  | ($s | length) as $len
  | if $len <= 3 then $s
    else ($s[0:$len-3] | commas) + "," + $s[$len-3:]
    end;
'

JQ_PROGRAM="$JQ_PAD_DEFS"$(cat <<'JQEOF'
# Renders a percentage as a right-justified, width-wide cell whose background
# is filled left-to-right in proportion to the value (100 -> whole cell,
# 50 -> half, etc.), like an inline progress bar behind the number. $useColor
# is off when stdout isn't a terminal (piped to a file, etc.) so saved output
# never contains raw escape codes. 44 = blue background -- change here to
# recolor.
def barcell(pct; w):
  (pct | tostring | lpad(w)) as $text
  | if $useColor then
      ((pct / 100 * w) | round) as $filled
      | ("[44m" + $text[0:$filled] + "[0m" + $text[$filled:])
    else
      $text
    end;

def tb:
  . / 1099511627776 * 100 | round / 100;

# A few packageType values, used as-is elsewhere, read poorly as a plain
# label in a customer-facing report; only the ones flagged so far get a
# nicer display name, and only wrapping if that pattern shows up again.
def displayname:
  {"HuggingFaceML": "Hugging Face ML", "Gitlfs": "Git LFS"} as $names
  | $names[.] // .;

# Integer percentages that always sum to exactly 100 (largest-remainder /
# Hamilton apportionment method), instead of naive per-row rounding which
# can drift away from 100 due to independent rounding errors.
def largest_remainder_pcts(sizes; total):
  sizes as $sizes
  | total as $tot
  | if $tot <= 0 then ($sizes | map(0))
    else
      ($sizes | map(. / $tot * 100)) as $raw
      | ($raw | map(floor)) as $floors
      | ($floors | add) as $flrSum
      | (100 - $flrSum) as $deficit
      | ([range(0; ($raw|length))] | map({i:., rem: ($raw[.] - $floors[.])})) as $withRem
      | ($withRem | sort_by(-.rem) | .[0:$deficit] | map(.i)) as $bumpIdx
      | ([range(0; ($raw|length))] | map(. as $idx | $floors[$idx] + (if ($bumpIdx | index($idx)) != null then 1 else 0 end)))
    end;

.rawcount as $cnt
| .dedupsize as $ds
| [$ds.lastyear, $ds.b1_3, $ds.b3_5, $ds.b5plus, $ds.never_lastyear, $ds.never_older] as $sizes
| ($sizes | add) as $grandTotal
| (largest_remainder_pcts($sizes; $grandTotal)) as $pcts
| [
    {c:"Last year",                  n:$cnt.lastyear,        s:$sizes[0], p:$pcts[0]},
    {c:"1-3 years ago",              n:$cnt.b1_3,            s:$sizes[1], p:$pcts[1]},
    {c:"3-5 years ago",              n:$cnt.b3_5,            s:$sizes[2], p:$pcts[2]},
    {c:"5+ years ago",               n:$cnt.b5plus,          s:$sizes[3], p:$pcts[3]},
    {c:"Never - created last year",  n:$cnt.never_lastyear,  s:$sizes[4], p:$pcts[4]},
    {c:"Never - created 1+ year ago", n:$cnt.never_older,    s:$sizes[5], p:$pcts[5]}
  ] as $rows
| ($rows | map(.n) | add) as $totalCount
| ($rows + [{c:"Total", n:$totalCount, s:$grandTotal, p:($pcts|add)}]) as $allrows
| ($allrows | map(. + {sizeTB: (.s|tb)})) as $displayRows
| ($displayRows | map(
    "\(.c|rpad(28)) | \((.n|tostring|commas)|lpad(14)) | \((.sizeTB|tostring)|lpad(17)) | "
    + (if .c == "Total" then (.p|tostring|lpad(19)) else barcell(.p; 19) end)
  )) as $formatted
| (
    ["\("LAST DOWNLOADED"|rpad(28)) | \("ARTIFACT COUNT"|lpad(14)) | \("BINARY SIZE (TB)"|lpad(17)) | \("% OF BINARY STORAGE"|lpad(19))",
     "\(dashes(28))-+-\(dashes(14))-+-\(dashes(17))-+-\(dashes(19))"]
    + $formatted[0:-1]
    + ["\(dashes(28))-+-\(dashes(14))-+-\(dashes(17))-+-\(dashes(19))"]
    + [$formatted[-1]]
  ) as $mainTable

| (.typeTotals) as $typeTotals
| (.typeCounts) as $typeCounts
| (.unknownRepos | sort) as $unknownRepos
| ($typeTotals | to_entries | sort_by(-.value) | .[0:5]) as $topEntries
# Percentage of the OVERALL deduplicated total, not forced to sum to 100 --
# the top 5 types are a subset, so (unlike the main table's buckets, which
# are an exhaustive partition) their share of storage can legitimately add
# up to less than 100.
| ($topEntries | map(
    {
      c: ((.key | displayname) + (if .key == "Unknown" and ($unknownRepos|length) > 0
                  then " (" + ($unknownRepos | join(", ")) + ")"
                  else "" end)),
      n: ($typeCounts[.key] // 0),
      sizeTB: (.value | tb),
      p: (if $grandTotal <= 0 then 0 else ((.value / $grandTotal * 100) | round) end)
    }
  )) as $typeRows
| ($typeRows | map("\(.c|rpad(28)) | \((.n|tostring|commas)|lpad(14)) | \((.sizeTB|tostring)|lpad(17)) | \(barcell(.p; 19))")) as $typeFormatted
| (
    ["\("TOP 5 REPO TYPE"|rpad(28)) | \("ARTIFACT COUNT"|lpad(14)) | \("BINARY SIZE (TB)"|lpad(17)) | \("% OF BINARY STORAGE"|lpad(19))",
     "\(dashes(28))-+-\(dashes(14))-+-\(dashes(17))-+-\(dashes(19))"]
    + $typeFormatted
  ) as $typeTable

| ($mainTable + ["", ""] + $typeTable)
| .[]
JQEOF
)

BINARIES_TB=$(awk -v b="$BINARIES_BYTES" 'BEGIN{printf "%.2f", b/1099511627776}')
# Left-justified labels (no leading whitespace); padded to the width of
# "Binary storage: " so both values still line up in the same column.
printf '%s\n' "${C_BOLD}📊 INSTANCE SUMMARY${C_RESET}"
printf "%-16s${C_BOLD}%s${C_RESET}\n" "Binary storage: " "${BINARIES_TB} TB"
printf "%-16s${C_BOLD}%s${C_RESET}\n" "Repositories: " "$REPO_COUNT"
printf "%-16s${C_BOLD}%s${C_RESET}\n" "Biggest repo: " "$(format_count "$BIGGEST_REPO_COUNT") artifacts ($BIGGEST_REPO_TYPE)"
echo

# Project each page to flat TSV in parallel (each to its OWN file, never a
# shared pipe -- concurrent writers on one pipe can interleave mid-line once
# a write exceeds the kernel's atomic-write threshold, silently corrupting
# records), then feed every projected file into the single awk pass above.
if [ "${#PAGE_FILES[@]}" -gt 0 ]; then
  printf '%s\n' "${PAGE_FILES[@]}" | xargs -P "$PARALLEL_JOBS" -I {} bash -c 'jq -r "$1" "$2" > "$2.tsv"' _ "$PROJECT_PROGRAM" {}
  PROJECTED_FILES=("${PAGE_FILES[@]/%/.tsv}")
  TMP_FILES+=("${PROJECTED_FILES[@]}")
else
  PROJECTED_FILES=()
fi

# </dev/null matters when PROJECTED_FILES is empty (no successful pages at
# all): with zero file arguments, awk falls back to reading stdin, which in
# this interactive script is the user's terminal -- without the redirect it
# would hang forever instead of producing an all-zero report.
SUMMARY_JSON=$(awk -v REPO_TYPES_FILE="$REPO_TYPES_FILE" -v cutoff1y="$CUTOFF_1Y" -v cutoff3y="$CUTOFF_3Y" -v cutoff5y="$CUTOFF_5Y" \
  -v CSV_FILE="${CSV_FILE:-}" \
  -f "$AWK_PROGRAM_FILE" ${PROJECTED_FILES[@]+"${PROJECTED_FILES[@]}"} </dev/null)

echo "$SUMMARY_JSON" | jq -r --argjson useColor "$USE_COLOR" "$JQ_PROGRAM"

# --- 7. Per-repo gap check against storageinfo's own per-repo ground truth ---
# storageinfo (fetched in step 3) already reports authoritative filesCount /
# usedSpaceInBytes per repo, computed server-side and independent of what
# this credential's own AQL queries could see. Comparing the two, repo by
# repo, turns "the totals are off by some %" into an exact, named list of
# which repos are short and by how much -- with zero extra API calls, since
# the ground truth was already fetched and the per-repo totals were already
# being accumulated in the awk pass above.
#
# A gap here means AQL (and the repository listing) are both filtering this
# credential out of that repo's contents. This can happen even with a full
# platform-admin credential, when the repo is assigned to a JFrog Project
# the account isn't a member of -- no further query recovers it, since that
# isolation is enforced by the same authorization layer AQL itself uses.
# Only adding the account to the owning project (or using a credential that
# already is a member) does.
REPO_GAP_MIN_BYTES=10485760 # 10 MB -- below this, treat as snapshot-timing noise, not a real gap
REPO_GAP_MAX_ROWS=10        # only the biggest gaps are worth showing
REPO_GAP_PCT_THRESHOLD=1    # only warn once the overall gap is worth acting on

BYREPO_JSON="$(echo "$SUMMARY_JSON" | jq -c '.byRepo // {}')"

# Repos with any raw shortfall above the floor -- a superset of what will
# actually be reported once the .evidence adjustment below (which can only
# shrink a gap, never grow one) is applied.
CANDIDATE_REPOS=$(jq -r \
  --rawfile gt "$REPO_GROUNDTRUTH_FILE" \
  --argjson byRepo "$BYREPO_JSON" \
  --argjson minBytes "$REPO_GAP_MIN_BYTES" -n '
  ($gt | rtrimstr("\n") | (if length == 0 then [] else split("\n") end)) as $lines
  | $lines[]
  | split("\t")
  | {repo: .[0], repoType: .[1], expBytes: (.[3] | tonumber)}
  | select(.repoType == "LOCAL" or .repoType == "CACHE" or .repoType == "FEDERATED")
  | select(.expBytes - ($byRepo[.repo].bytes // 0) >= $minBytes)
  | .repo
')

# JFrog Evidence (SBOMs, test results, signatures, build attestations) is
# stored under a reserved ".evidence" folder inside whatever repo the
# subject artifact lives in, but is categorically excluded from AQL's items
# domain -- even an explicit path match for ".evidence*" returns zero
# results, regardless of credential. It's not an access gap, so it would
# otherwise show up as one forever with no fix possible via AQL. The
# per-repo storage listing API can still reach it directly, and scoping the
# check to just the "<repo>/.evidence" subtree (rather than the whole repo)
# keeps it cheap -- so it's only done for the bounded, usually small set of
# repos that already show a raw gap above the floor.
EVIDENCE_ADJUST_FILE=$(mktemp)
TMP_FILES+=("$EVIDENCE_ADJUST_FILE")
: >"$EVIDENCE_ADJUST_FILE"
if [ -n "$CANDIDATE_REPOS" ]; then
  # One HTTP call per candidate repo, run with the same concurrency as the
  # main fallback -- this list is usually small, but running it fully
  # sequential can still add double-digit seconds on its own. Each worker
  # writes its own file (never a shared pipe -- see the identical reasoning
  # at the page-projection step above). Reuses $OUT_DIR rather than a
  # separate temp dir, since that's already unconditionally removed by the
  # cleanup trap on exit, including an interrupted run; a distinct filename
  # prefix keeps these from ever colliding with the fallback's own *.json
  # page files there.
  #
  # Announced explicitly -- this makes real HTTP calls, so without a message
  # here the script otherwise goes silent for a few seconds right after the
  # last table prints, which reads like a hang rather than ongoing work.
  echo
  step "Checking data completeness"
  export AUTH_USER AUTH_PASS ARTIFACTORY_URL
  mkdir -p "$OUT_DIR"
  printf '%s\n' "$CANDIDATE_REPOS" | xargs -P "$PARALLEL_JOBS" -I {} bash -c '
    repo="$1"
    resp=$(curl -sS -u "$AUTH_USER:$AUTH_PASS" "$ARTIFACTORY_URL/api/storage/${repo}/.evidence?list&deep=1")
    if echo "$resp" | jq -e "has(\"files\")" >/dev/null 2>&1; then
      echo "$resp" | jq -r --arg repo "$repo" "[\$repo, (.files|length), ([.files[].size] | add // 0)] | @tsv"
    fi
  ' _ {} >"$OUT_DIR/evidence__{}.tsv" 2>/dev/null
  cat "$OUT_DIR"/evidence__*.tsv >"$EVIDENCE_ADJUST_FILE" 2>/dev/null || true
  step_done "done"
  echo "🐸"
  echo
fi

GAP_REPORT=$(jq -n -r \
  --rawfile gt "$REPO_GROUNDTRUTH_FILE" \
  --argjson byRepo "$BYREPO_JSON" \
  --rawfile ev "$EVIDENCE_ADJUST_FILE" \
  --argjson minBytes "$REPO_GAP_MIN_BYTES" \
  --argjson maxRows "$REPO_GAP_MAX_ROWS" \
  --argjson pctThreshold "$REPO_GAP_PCT_THRESHOLD" \
  --argjson binariesBytes "$BINARIES_BYTES" \
  "$JQ_PAD_DEFS"'
  def humansize:
    . as $b
    | if $b >= 1099511627776 then (($b/1099511627776*100 | round/100 | tostring) + " TB")
      elif $b >= 1073741824 then (($b/1073741824*100 | round/100 | tostring) + " GB")
      elif $b >= 1048576 then (($b/1048576*100 | round/100 | tostring) + " MB")
      else (($b | tostring) + " B")
      end;

  ($ev | rtrimstr("\n") | (if length == 0 then [] else split("\n") end)
   | map(split("\t") | {key: .[0], value: {count: (.[1]|tonumber), bytes: (.[2]|tonumber)}}) | from_entries
  ) as $evidence
  | ($gt | rtrimstr("\n") | (if length == 0 then [] else split("\n") end)) as $lines
  | [
      $lines[]
      | split("\t")
      | {repo: .[0], repoType: .[1], expCount: (.[2] | tonumber), expBytes: (.[3] | tonumber), projectKey: .[4]}
      | select(.repoType == "LOCAL" or .repoType == "CACHE" or .repoType == "FEDERATED")
      | . + {evidenceCount: ($evidence[.repo].count // 0), evidenceBytes: ($evidence[.repo].bytes // 0)}
      | .expCount -= .evidenceCount
      | .expBytes -= .evidenceBytes
      | . + {foundCount: ($byRepo[.repo].count // 0), foundBytes: ($byRepo[.repo].bytes // 0)}
      | . + {missing: (.expBytes - .foundBytes)}
      | select(.missing >= $minBytes)
    ] as $gaps
  | ($gaps | map(.missing) | add // 0) as $totalMissing
  | (if $binariesBytes > 0 then ($totalMissing / $binariesBytes * 100) else 0 end) as $gapPct
  | if ($gaps | length) == 0 or $gapPct <= $pctThreshold then
      ""
    else
      ($gaps | sort_by(-.missing) | .[0:$maxRows]) as $shown
      | [
          "",
          "WARNING: \($gaps | length) repo(s) inaccessible to this credential (~\($totalMissing | humansize), \(($gapPct*10 | round)/10)% of storage) -- add the account to the owning JFrog Project",
          "",
          "\("REPO" | rpad(44)) | \("PROJECT" | rpad(14)) | \("EXPECTED FILES" | lpad(14)) | \("FOUND FILES" | lpad(11)) | \("MISSING" | lpad(10))",
          "\(dashes(44))-+-\(dashes(14))-+-\(dashes(14))-+-\(dashes(11))-+-\(dashes(10))"
        ]
        + ($shown | map(
            "\(.repo | rpad(44)) | \(.projectKey | rpad(14)) | \((.expCount | tostring | commas) | lpad(14)) | \((.foundCount | tostring | commas) | lpad(11)) | \(.missing | humansize | lpad(10))"
          ))
      | .[]
    end
')

if [ -n "$GAP_REPORT" ]; then
  echo "$GAP_REPORT"
fi

if [ "$CSV_EXPORT" = "yes" ]; then
  echo
  step "Exporting raw per-artifact data to CSV"
  CSV_ACTUAL_BYTES=$(wc -c <"$CSV_FILE" 2>/dev/null | tr -d ' ')

  # Applied on the file's real size, not the rough pre-fetch estimate above --
  # a CSV of repeated repo paths, package types, and fixed-width columns
  # typically compresses very well, and this only matters once the actual
  # size is known. gzip is optional here (best-effort, not a hard prereq):
  # skipped silently if it isn't on PATH. Automatic, not confirmed -- gzip is
  # cheap and reversible (gunzip recovers the exact original), so it isn't
  # worth an interactive prompt the way a destructive or slow step would be.
  # CSV_FILE is repointed at the .gz so only the final artifact is reported
  # below, not both the pre- and post-compression sizes.
  CSV_GZIP_THRESHOLD_BYTES=$((20 * 1024 * 1024)) # 20 MB
  if [ -n "$CSV_ACTUAL_BYTES" ] && [ "$CSV_ACTUAL_BYTES" -gt "$CSV_GZIP_THRESHOLD_BYTES" ] 2>/dev/null && command -v gzip >/dev/null 2>&1; then
    gzip -f "$CSV_FILE"
    CSV_FILE="${CSV_FILE}.gz"
    CSV_ACTUAL_BYTES=$(wc -c <"$CSV_FILE" 2>/dev/null | tr -d ' ')
  fi
  step_done "$CSV_FILE ($(human_bytes "${CSV_ACTUAL_BYTES:-0}"))"
fi
