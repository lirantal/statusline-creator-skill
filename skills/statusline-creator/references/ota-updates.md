# OTA Updates for Statuslines

If you are distributing your statusline to other users (published as a repo, shared
as a package), they need a way to receive updates without manually pulling and
reinstalling. This document describes an OTA (over-the-air) update pattern
that runs in the background on each Claude Code session start.

For personal-use statuslines, skip this entirely.

---

## Architecture overview

```
SessionStart hook
  └── ~/.claude/update.sh  (background, singleton-locked)
        ├── rate limit check (1/hour)
        ├── check GitHub releases API
        ├── download tarball + checksums.sha256
        ├── verify SHA256 (fail-closed)
        ├── validate scripts (bash -n, shebang)
        ├── backup current files
        ├── atomic swap (cp .tmp → mv)
        ├── health check → rollback on failure
        └── write notification for statusline badge
```

The statusline itself stays read-only. It never checks for updates, downloads
anything, or mutates files. It only reads a notification file to show "Updated"
or "Available" badges.

---

## SessionStart hook registration

Claude Code runs hooks on session start. Register the update check as a background
process so it never blocks startup:

```json
{
  "hooks": {
    "SessionStart": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "~/.claude/update.sh >/dev/null 2>&1 &",
            "timeout": 5
          }
        ]
      }
    ]
  }
}
```

The `&` and redirect ensure the updater runs in the background. Alternatively, use
the native `async: true` hook property for cleaner background execution:

```json
{
  "hooks": [
    {
      "type": "command",
      "command": "~/.claude/update.sh",
      "async": true
    }
  ]
}
```

Either approach works. The shell `&` pattern is compatible with older Claude Code
versions; `async: true` is the idiomatic approach.

In your `install.sh`, merge this hook into existing settings without clobbering
other hooks:

```bash
UPDATE_HOOK="~/.claude/update.sh >/dev/null 2>&1 &"
jq --arg hook "$UPDATE_HOOK" '
    .hooks.SessionStart //= [] |
    if (.hooks.SessionStart | map(select(.hooks[]?.command == $hook)) | length) == 0 then
        .hooks.SessionStart += [{"hooks": [{"type": "command", "command": $hook, "timeout": 5}]}]
    else . end
' "$SETTINGS_FILE" > "${SETTINGS_FILE}.tmp" \
    && mv "${SETTINGS_FILE}.tmp" "$SETTINGS_FILE"
```

---

## Singleton locking

Only one update check should run at a time. Use `mkdir` (POSIX-atomic) with
stale lock recovery:

```bash
LOCK_DIR="${CACHE_DIR}/update.lock"
LOCK_TTL=600  # reclaim after 10 minutes

if ! mkdir "$LOCK_DIR" 2>/dev/null; then
    if [[ -d "$LOCK_DIR" ]]; then
        lock_age=$(( $(date +%s) - $(stat -f %m "$LOCK_DIR" 2>/dev/null \
                                  || stat -c %Y "$LOCK_DIR" 2>/dev/null \
                                  || echo 0) ))
        if (( lock_age > LOCK_TTL )); then
            rm -rf "$LOCK_DIR"
            mkdir "$LOCK_DIR" 2>/dev/null || exit 0
        else
            exit 0
        fi
    else
        exit 0
    fi
fi
trap 'rm -rf "$LOCK_DIR"' EXIT
```

The `stat` call uses `-f %m` (macOS) with `-c %Y` (Linux) fallback.

---

## Rate limiting

Avoid hammering the GitHub API. Write the check timestamp to a file and skip if
less than 1 hour has passed:

```bash
LAST_CHECK_FILE="${CACHE_DIR}/last_update_check"

if [[ -f "$LAST_CHECK_FILE" ]]; then
    last_check=$(cat "$LAST_CHECK_FILE" 2>/dev/null || echo 0)
    now=$(date +%s)
    if (( now - last_check < 3600 )); then
        exit 0
    fi
fi

# After a successful API call:
date +%s > "$LAST_CHECK_FILE"
```

---

## Version detection

Embed the version in a comment on line 2 of every script:

```bash
#!/bin/bash
# my-statusline v1.2.3: Description of what this does
```

Extract it with:

```bash
get_installed_version() {
    sed -n '2s/.*v\([0-9.]*\).*/\1/p' "${CLAUDE_DIR}/statusline-command.sh" 2>/dev/null
}
```

Compare versions with a simple semver function:

```bash
semver_gt() {
    local IFS=.
    local i a=($1) b=($2)
    for ((i=0; i<${#a[@]}; i++)); do
        local av=${a[i]:-0} bv=${b[i]:-0}
        if (( av > bv )); then return 0; fi
        if (( av < bv )); then return 1; fi
    done
    return 1
}
```

---

## Checking for updates

Use `gh` CLI first (handles private repos), fall back to `curl`:

```bash
check_update() {
    local current_ver="$1"
    local api_response=""

    if command -v gh &>/dev/null; then
        api_response=$(gh api "repos/${REPO}/releases/latest" 2>/dev/null) || true
    fi
    if [[ -z "$api_response" ]]; then
        api_response=$(curl --proto '=https' --tlsv1.2 --fail --silent --max-time 10 \
            "https://api.github.com/repos/${REPO}/releases/latest" 2>/dev/null) || return 1
    fi

    local latest_ver
    latest_ver=$(echo "$api_response" | jq -r '.tag_name // empty' 2>/dev/null | sed 's/^v//')
    [[ -z "$latest_ver" ]] && return 1

    semver_gt "$latest_ver" "$current_ver" || return 1

    # Prefer uploaded release assets over auto-generated tarballs
    local tarball_url checksum_url=""
    tarball_url=$(echo "$api_response" | jq -r '
        .assets[]? | select(.name | test("\\.tar\\.gz$")) | .browser_download_url
    ' 2>/dev/null | head -1)
    checksum_url=$(echo "$api_response" | jq -r '
        .assets[]? | select(.name | test("checksums\\.sha256$")) | .browser_download_url
    ' 2>/dev/null | head -1)

    # Fallback to auto-generated tarball
    [[ -z "$tarball_url" ]] && tarball_url=$(echo "$api_response" | jq -r '.tarball_url // empty' 2>/dev/null)
    [[ -z "$tarball_url" ]] && return 1

    echo "${latest_ver}|${tarball_url}|${checksum_url}"
}
```

---

## Domain pinning

Before downloading any URL, validate the host against an allowlist. This adds a
layer of defense against tampered release metadata pointing to unexpected hosts
(note: this checks the initial URL, not post-redirect targets):

```bash
ALLOWED_HOSTS="github.com api.github.com"

_validate_url() {
    local url="$1"
    local host
    host=$(echo "$url" | sed -n 's|^https\{0,1\}://\([^/:]*\).*|\1|p')
    for allowed in $ALLOWED_HOSTS; do
        [[ "$host" == "$allowed" ]] && return 0
    done
    return 1
}
```

Call `_validate_url` before every download.

---

## SHA256 checksum verification

This is the most important security control. **Fail closed**: if there is no
checksum, or it cannot be downloaded, or it doesn't match, abort the update.
Never fall back to an unverified tarball.

```bash
# Abort if no checksum URL in release
if [[ -z "$checksum_url" ]]; then
    log "Aborted: release has no checksum asset"
    rm -rf "$STAGING_DIR"
    return 1
fi

# Download and verify
local checksum_file="${STAGING_DIR}/checksums.sha256"
_download "$checksum_url" "$checksum_file" || { rm -rf "$STAGING_DIR"; return 1; }

local expected_hash actual_hash tarball_name
tarball_name=$(basename "$tarball")
expected_hash=$(grep "$tarball_name" "$checksum_file" | grep -o '^[a-f0-9]\{64\}' | head -1)
actual_hash=$(shasum -a 256 "$tarball" 2>/dev/null || sha256sum "$tarball" 2>/dev/null)
actual_hash=$(echo "$actual_hash" | cut -d' ' -f1)

if [[ -z "$expected_hash" ]] || [[ "$actual_hash" != "$expected_hash" ]]; then
    log "Aborted: checksum mismatch"
    rm -rf "$STAGING_DIR"
    return 1
fi
```

On the release side, generate the checksum file with:

```bash
# macOS uses shasum, Linux uses sha256sum
(shasum -a 256 my-statusline-v1.2.3.tar.gz 2>/dev/null \
    || sha256sum my-statusline-v1.2.3.tar.gz) > checksums.sha256
```

Upload both the tarball and `checksums.sha256` as release assets.

---

## Script validation

Before applying, verify that downloaded scripts are syntactically valid and have
the expected shebang:

```bash
for f in "${required_files[@]}"; do
    if ! bash -n "${extracted}/${f}" 2>/dev/null; then
        log "Validation failed: syntax error in ${f}"
        rm -rf "$STAGING_DIR"; return 1
    fi
    local shebang
    shebang=$(head -1 "${extracted}/${f}")
    if [[ "$shebang" != "#!/bin/bash" ]] && [[ "$shebang" != "#!/usr/bin/env bash" ]]; then
        log "Validation failed: bad shebang in ${f}"
        rm -rf "$STAGING_DIR"; return 1
    fi
done
```

---

## Atomic apply with backup and rollback

The apply sequence: backup, mark in-progress, swap files, health check, clean up.
If anything fails, roll back from the backup.

```bash
apply_update() {
    local version="$1"
    local current_ver=$(get_installed_version)

    # 1. Backup
    local backup_dir="${CACHE_DIR}/versions/v${current_ver}"
    mkdir -p "$backup_dir"
    for f in "${OTA_FILES[@]}"; do
        [[ -f "${CLAUDE_DIR}/${f}" ]] && cp "${CLAUDE_DIR}/${f}" "$backup_dir/"
    done

    # 2. Mark in-progress (crash recovery)
    local marker="${CACHE_DIR}/apply_in_progress"
    printf '%s\n%s' "$version" "$current_ver" > "$marker"

    # 3. Atomic swap each file: cp to .tmp, then mv (new inode)
    for f in "${OTA_FILES[@]}"; do
        if [[ -f "${STAGING_DIR}/${f}" ]]; then
            cp "${STAGING_DIR}/${f}" "${CLAUDE_DIR}/${f}.tmp"
            chmod +x "${CLAUDE_DIR}/${f}.tmp"
            mv -f "${CLAUDE_DIR}/${f}.tmp" "${CLAUDE_DIR}/${f}"
        fi
    done

    # 4. Health check
    local new_ver=$(get_installed_version)
    if [[ "$new_ver" != "$version" ]]; then
        # Rollback
        for f in "${OTA_FILES[@]}"; do
            [[ -f "${backup_dir}/${f}" ]] && cp "${backup_dir}/${f}" "${CLAUDE_DIR}/${f}"
        done
        rm -f "$marker"
        return 1
    fi

    # 5. Clean up
    rm -rf "$STAGING_DIR"
    rm -f "$marker"
    printf '%s\n%s' "$version" "$(date +%s)" > "${CACHE_DIR}/update_notification"
}
```

The `mv` trick (cp to `.tmp` then `mv`) is safe even for the currently running script
because `mv` creates a new inode. The shell keeps the old file descriptor open and
continues executing from the old content.

---

## Crash recovery

If the updater is killed mid-apply, the `apply_in_progress` marker survives. On the
next run, detect it and roll back before doing anything else:

```bash
_marker="${CACHE_DIR}/apply_in_progress"
if [[ -f "$_marker" ]]; then
    _backup_ver=$(tail -1 "$_marker" 2>/dev/null)
    _backup_dir="${CACHE_DIR}/versions/v${_backup_ver}"
    if [[ -d "$_backup_dir" ]]; then
        for f in "${OTA_FILES[@]}"; do
            [[ -f "${_backup_dir}/${f}" ]] && cp "${_backup_dir}/${f}" "${CLAUDE_DIR}/${f}"
        done
    fi
    rm -f "$_marker"
fi
```

---

## Statusline badge

Show a transient badge in the statusline when an update was applied or is available.
The statusline reads notification files (no network I/O):

```bash
# In your statusline.sh, near the end of output:
_notif="${CACHE_DIR}/update_notification"
_avail="${CACHE_DIR}/update_available"

if [[ -f "$_notif" ]]; then
    _ver=$(head -1 "$_notif" 2>/dev/null)
    _time=$(tail -1 "$_notif" 2>/dev/null)
    # Show badge for 1 hour after update; update.sh cleans up stale files on next run
    if (( $(date +%s) - _time < 3600 )); then
        printf ' %s🔄 Updated to v%s%s' "$GREEN" "$_ver" "$R"
    fi
elif [[ -f "$_avail" ]]; then
    _ver=$(head -1 "$_avail" 2>/dev/null)
    printf ' %s🔄 v%s available%s' "$YELLOW" "$_ver" "$R"
fi
```

The notification auto-expires after 1 hour (update.sh cleans up stale files on its
next run). The "available" badge persists until the user applies the update.

---

## Update modes

Support two modes via an env var:

| Mode | Behavior |
|---|---|
| `auto` (default) | Download, verify, and apply immediately |
| `notify` | Download, verify, and stage. Show badge. User applies manually. |

```bash
UPDATE_MODE="${TOOL_AUTO_UPDATE:-auto}"

if [[ "$UPDATE_MODE" == "notify" ]]; then
    printf '%s\n%s' "$version" "$STAGING_DIR" > "${CACHE_DIR}/update_available"
else
    apply_update "$version"
fi
```

---

## Security checklist

- `umask 077` at the top of update.sh (private staging/cache files)
- `chmod 700` on cache and state directories
- Domain pinning (only download from `github.com` / `api.github.com`)
- TLS enforcement on curl (`--proto '=https' --tlsv1.2`)
- Fail-closed checksums (no checksum = no update, never skip)
- Script validation before apply (`bash -n` + shebang check)
- Atomic file operations throughout (no partial state visible)
- No `eval` or `source` on downloaded content before validation

---

## Publishing releases

On the release side, build a tarball and checksum, then upload both as GitHub
release assets:

```bash
# Build
tar czf my-statusline-v1.2.3.tar.gz statusline.sh update.sh install.sh

# Checksum
shasum -a 256 my-statusline-v1.2.3.tar.gz > checksums.sha256

# Publish
gh release create v1.2.3 \
    --title "v1.2.3" \
    --generate-notes \
    my-statusline-v1.2.3.tar.gz \
    checksums.sha256
```

Before publishing, verify:
- Working tree is clean (`git status --porcelain`)
- Tag doesn't already exist
- Version strings are consistent across all files
- `bash -n` passes on all scripts in the tarball

---

## Reference implementation

[claude-pulse](https://github.com/omriariav/claude-pulse) implements this pattern.
See `update.sh` and `release.sh` for a working example with automated tests
covering the OTA flow.
