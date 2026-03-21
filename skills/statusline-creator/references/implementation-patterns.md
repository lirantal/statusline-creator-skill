# Statusline Implementation Patterns

## Color constants

Always define colors as `$'...'` bash literals — this correctly interprets `\033`:

```bash
R=$'\033[0m'              # reset
RED=$'\033[38;2;255;85;85m'
ORANGE=$'\033[38;2;255;165;0m'
YELLOW=$'\033[38;2;255;215;0m'
GREEN=$'\033[38;2;80;250;123m'
BLUE=$'\033[38;2;100;170;255m'
DIM=$'\033[38;2;128;128;128m'
WHITE=$'\033[38;2;220;220;220m'
SEP="${DIM}│${R}"
```

Never define them as regular strings (`RED="\033[...]"`) — `\033` won't be
interpreted and raw escape sequences will appear in the output.

---

## Cache and lock file naming

Hash the project path to create stable, per-project cache keys:

```bash
CACHE_DIR="${XDG_CACHE_HOME:-$HOME/.cache}/<tool>-statusline"
mkdir -p "$CACHE_DIR"
PATH_HASH=$(printf '%s' "$GIT_ROOT" | cksum | cut -d' ' -f1)

CACHE_FILE="$CACHE_DIR/${PATH_HASH}.json"
LOCK_DIR="$CACHE_DIR/${PATH_HASH}.lock"    # directory used as atomic lock
ERR_FILE="$CACHE_DIR/${PATH_HASH}.err"
NOSCAN_FILE="$CACHE_DIR/${PATH_HASH}.noscan"  # written on exit code 3
```

For multiple scan types, namespace with a suffix:
```bash
SCA_CACHE="$CACHE_DIR/${PATH_HASH}.sca.json"
SAST_CACHE="$CACHE_DIR/${PATH_HASH}.sast.json"
# etc.
```

---

## Cache age helper

```bash
file_age() {
    local f="$1"
    [[ -f "$f" ]] || { printf '999999'; return; }
    local mtime
    mtime=$(stat -c %Y "$f" 2>/dev/null \
         || stat -f %m "$f" 2>/dev/null \  # macOS fallback
         || printf '0')
    printf '%d' $(( $(date +%s) - mtime ))
}

# For a scan that has both a result cache and a noscan sentinel:
scan_age() {
    local cache="$1" noscan="$2"
    local ca na
    ca=$(file_age "$cache")
    na=$(file_age "$noscan")
    (( ca < na )) && printf '%d' "$ca" || printf '%d' "$na"
}
```

---

## Background scan with atomic locking

`mkdir` is POSIX-atomic — use it for locking to prevent concurrent scans
across multiple terminal tabs or Claude windows on the same project.

```bash
trigger_scan_bg() {
    mkdir "$LOCK_DIR" 2>/dev/null || return  # exit silently if already running

    (
        trap 'rm -rf "$LOCK_DIR"' EXIT  # always release lock on exit

        cd "$GIT_ROOT"
        local tmp="$CACHE_FILE.tmp" exit_code=0

        # Capture exit code explicitly — never use `|| true` which loses it
        your_scan_command --json > "$tmp" 2>"$ERR_FILE" || exit_code=$?

        if (( exit_code == 3 )); then
            # No supported project/manifest — write sentinel, don't retry every render
            printf '3' > "$NOSCAN_FILE"
            rm -f "$tmp"
        elif [[ -s "$tmp" ]] && jq -e '.expected_key' "$tmp" &>/dev/null; then
            # Valid result (exit 0 = clean, exit 1 = findings — both are valid)
            rm -f "$NOSCAN_FILE"       # clear stale sentinel if project changed
            mv "$tmp" "$CACHE_FILE"
        else
            rm -f "$tmp"               # partial/unrecognised output — leave state unchanged
        fi
    ) &>/dev/null &
    disown  # detach from current shell so it survives script exit
}
```

Trigger the scan when the cache is stale:
```bash
AGE=$(scan_age "$CACHE_FILE" "$NOSCAN_FILE")
(( AGE > CACHE_TTL )) && trigger_scan_bg
SCANNING=$( [[ -d "$LOCK_DIR" ]] && printf 'true' || printf 'false' )
```

---

## Exit code semantics

Most security CLI tools follow this convention:

| Exit code | Meaning | Action |
|---|---|---|
| `0` | Success, no findings | Write result to cache |
| `1` | Findings present (still valid JSON) | Write result to cache |
| `2` | Scan error / failure | Log to `.err`; leave cache unchanged |
| `3` | No supported project/files | Write `.noscan` sentinel |

Exit code 3 is the critical case — if you treat it like exit 2, the script will
spin forever retrying a scan that can never succeed. Always write the sentinel.

---

## State machine for each scan slot

Every scan slot has exactly five states. Check them in this order:

```
1. SCANNING  (lock dir exists, no cache) → show "scanning..."
2. NOSCAN    (sentinel exists, no cache) → show "no supported files"
3. AUTH_ERR  (err file has auth keywords, no cache) → show "⚠ auth required"
4. INITIALIZING (nothing exists yet)    → show "initializing..."
5. HAS_RESULT  (cache file exists)      → parse and show result
                                           (spinner if SCANNING too)
```

```bash
build_segment() {
    if $SCANNING && [[ ! -f "$CACHE_FILE" ]]; then
        printf '%sscanning...%s' "$DIM" "$R"; return
    fi
    if [[ -f "$NOSCAN_FILE" ]] && [[ ! -f "$CACHE_FILE" ]]; then
        printf '%sno supported files%s' "$DIM" "$R"; return
    fi
    if [[ -s "$ERR_FILE" ]] && grep -qi 'auth\|token\|login' "$ERR_FILE" 2>/dev/null \
       && [[ ! -f "$CACHE_FILE" ]]; then
        printf '%s⚠ auth required%s' "$ORANGE" "$R"; return
    fi
    if [[ ! -f "$CACHE_FILE" ]]; then
        printf '%sinitializing...%s' "$DIM" "$R"; return
    fi
    # parse cache and build result segment
}
```

---

## Single jq call pattern

Make one `jq` call per cache file — multiple calls on the same file multiply latency.
Use `IFS='|' read -r` to split the result into shell variables:

```bash
# General pattern: extract N fields in one call, pipe-delimited
IFS='|' read -r FIELD1 FIELD2 FIELD3 < <(
    jq -r '[
        (.some_field // "default" | tostring),
        (.count_field // 0 | tostring),
        ([.items[]? | select(.type == "x")] | length | tostring)
    ] | join("|")' "$CACHE_FILE" 2>/dev/null || printf 'default|0|0'
)
```

Always provide a fallback (`|| printf '...'`) in case the cache file is malformed.

### Example: Snyk dependency scan (JSON output)

```bash
IFS='|' read -r OK TOTAL C H M L FIXABLE < <(
    jq -r '[
        (.ok // false | tostring),
        (.uniqueCount // 0 | tostring),
        ([.vulnerabilities[]? | select(.severity == "critical")] | length | tostring),
        ([.vulnerabilities[]? | select(.severity == "high")]     | length | tostring),
        ([.vulnerabilities[]? | select(.severity == "medium")]   | length | tostring),
        ([.vulnerabilities[]? | select(.severity == "low")]      | length | tostring),
        ([.vulnerabilities[]? | select(.isUpgradable == true or .isPatchable == true)] | length | tostring)
    ] | join("|")' "$CACHE_FILE" 2>/dev/null || printf 'false|0|0|0|0|0|0'
)
```

### Example: Snyk SAST scan (SARIF 2.1.0 output)

SARIF is a standard format used by many static analysis tools. Severity lives in
`results[].level` rather than a `severity` field: `error`=High, `warning`=Medium, `note`=Low.

```bash
IFS='|' read -r TOTAL H M L FIXABLE < <(
    jq -r '[
        (.runs[0].results | length | tostring),
        ([.runs[0].results[]? | select(.level == "error")]   | length | tostring),
        ([.runs[0].results[]? | select(.level == "warning")] | length | tostring),
        ([.runs[0].results[]? | select(.level == "note")]    | length | tostring),
        ([.runs[0].results[]? | select(.properties.isAutofixable == true)] | length | tostring)
    ] | join("|")' "$CACHE_FILE" 2>/dev/null || printf '0|0|0|0|0'
)
```

---

## Age formatting

```bash
fmt_age() {
    local s=$1
    if   (( s < 60 ));   then printf '%ds' "$s"
    elif (( s < 3600 )); then printf '%dm' "$(( s / 60 ))"
    else                      printf '%dh' "$(( s / 3600 ))"
    fi
}
```

When showing age of multiple independent scans, show the **oldest** (maximum age).
This is the most conservative signal — the user knows the least-fresh data point.

---

## Testing by seeding the cache

Never rely on live scans to test states. Seed the cache directly:

```bash
CACHE_DIR="$HOME/.cache/<tool>-statusline"
HASH=$(printf '%s' "$(git rev-parse --show-toplevel)" | cksum | cut -d' ' -f1)

# State: no issues
echo '{"ok":true,"uniqueCount":0,"vulnerabilities":[],"projectName":"my-app","packageManager":"npm"}' \
  > "$CACHE_DIR/${HASH}.sca.json"

# State: findings present
cp /path/to/real/snyk-output.json "$CACHE_DIR/${HASH}.sca.json"

# State: no supported manifest
printf '3' > "$CACHE_DIR/${HASH}.sca.noscan"
rm -f "$CACHE_DIR/${HASH}.sca.json"

# State: scanning in progress
mkdir "$CACHE_DIR/${HASH}.sca.lock"
rm -f "$CACHE_DIR/${HASH}.sca.json" "$CACHE_DIR/${HASH}.sca.noscan"
echo '{}' | ./statusline.sh
rmdir "$CACHE_DIR/${HASH}.sca.lock"

# Inspect raw ANSI output
echo '{}' | ./statusline.sh | cat -v
# Correct: ^[[38;2;255;85;85mC:2^[[0m
# Wrong:   \033[38;2;255;85;85mC:2\033[0m  (escapes not interpreted)
```

---

## Snyk-specific notes

**`snyk test` output** (JSON):
- `ok` — boolean, true if no vulnerabilities
- `uniqueCount` — total unique vulnerabilities
- `vulnerabilities[].severity` — `critical` / `high` / `medium` / `low`
- `vulnerabilities[].isUpgradable` — fix available via package upgrade
- `vulnerabilities[].isPatchable` — fix available via snyk patch
- `projectName` — name of the scanned project

**`snyk code test` output** (SARIF 2.1.0):
- `runs[0].results[]` — array of findings
- `results[].level` — `error` (High), `warning` (Medium), `note` (Low)
- `results[].properties.isAutofixable` — boolean, Snyk can generate a fix
- No "critical" level — Snyk Code tops out at High

**Exit code 3** for both commands means no supported project files were found.
Exit code 1 means findings were found — the output is still valid JSON, not an error.
