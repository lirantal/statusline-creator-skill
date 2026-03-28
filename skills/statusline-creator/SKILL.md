---
name: statusline-creator
description: >
  Builds a Claude Code statusline from scratch or extends an existing one. A statusline
  is a live status bar that runs a shell script after every Claude response and displays
  data — security scans, git status, API quotas, build state — directly in the session.
  Use this skill when the user says "create a statusline", "make a statusline for X",
  "add X to my statusline", "show Y in the status bar", "build a statusline plugin",
  or "I want live data in my Claude session". Trigger even if the user just describes
  wanting live feedback during a Claude session without naming "statusline" explicitly.
  The skill drives from idea to a working, installed, tested statusline without requiring
  the user to direct every step.
license: Apache-2.0
compatibility: >
  Requires bash and jq. Claude Code must be installed and authenticated. The statusline
  script runs in the project working directory — any data accessible from bash (CLI tools,
  git, file system, HTTP APIs) can be used as a data source. External tools (e.g. snyk,
  gh, kubectl) must be installed separately.
metadata:
  author: lirantal
  version: 1.0.0
---

# Statusline Creator

# Instructions

A Claude Code statusline is a bash script that Claude runs after every response. It
reads session metadata on stdin and prints a colored line to stdout — giving you live,
contextual data without leaving the session.

The key design challenge: the statusline must render **instantly**, every time.
There are two valid execution models — choose based on measured speed:

- **Inline execution**: run the command directly in the script. Only viable if the
  operation reliably completes in under ~300ms (fast HTTP endpoints, local file reads,
  cheap shell commands). Must be benchmarked first — never assume an operation is fast.
- **Background + cache**: the script always renders from cache; a background process
  refreshes it on a TTL. Required for any operation that is slow, variable, or
  potentially blocking (CLI scanners, external APIs, network calls to remote services).

**Before choosing a model, benchmark the operation** — see Step 1.

Read `references/protocol.md` at the start of every session — it covers the stdin
JSON schema, the settings.json format, and ANSI output rules. The authoritative
reference for the protocol is the official Claude Code statusline documentation at
https://code.claude.com/docs/en/statusline.md — check it for new fields or behaviors
not yet reflected in the reference files.
Read `references/implementation-patterns.md` when writing or debugging the script —
it covers the caching pattern, exit code handling, state machine, and jq recipes.
Read `references/ota-updates.md` if the user wants to distribute their statusline
to others with automatic updates — it covers the SessionStart hook pattern,
SHA256 checksum verification, atomic apply with rollback, and the release workflow.

---

### Step 1: Understand what the user wants to show

Ask the user what data source they want in their statusline. Common categories:
- **Security tools** — `snyk test`, `trivy`, `semgrep`, SAST/SCA/IaC scanners
- **Git/project state** — branch, dirty files, CI status, PR state
- **Resource usage** — API quotas, context window, cost, rate limits
- **Build/test state** — last test run result, coverage, lint status
- **Cloud/infra** — Kubernetes pod state, deploy status, environment

For each data source, establish:
1. What CLI command or HTTP request produces the data?
2. **Benchmark its actual runtime** — run it 3 times and record the wall-clock time:
   ```bash
   time <command>   # run 3× and note the range
   ```
   - Under ~300ms consistently → inline execution is viable
   - Over ~300ms, variable, or potentially blocking → use background + cache
3. What output format does it produce? (JSON, plain text, exit code only?)
4. What states should the statusline show? (clean, issues found, scanning, unavailable, auth error)

**Done when:** You have a concrete data source, its command, its output format, and a list of states.

---

### Step 2: Design the output line

Plan the display format before writing any code. A good statusline line has:
- A **label** identifying the tool (e.g. `🔒 snyk`, `🌿 git`, `☁ k8s`)
- **Segments** separated by `│` — one per logical data group
- **Color-coded counts or status** — use severity/urgency to drive color choice
- A **scan age** indicator when results come from a cache (`· 5m ago`)
- A **spinner** `⟳` when a background refresh is running

Design the full set of states the script will display:
```
label │ <primary result> │ <secondary result> │ <project> · <age> ⟳
label │ ✔ clean │ project · 2m ago
label │ scanning...
label │ no supported files
label │ ⚠ auth required  run: <cmd>
```

Use ANSI RGB colors. Standard palette for security tools (see `references/implementation-patterns.md`):
- Critical → Red `\033[38;2;255;85;85m`
- High → Orange `\033[38;2;255;165;0m`
- Medium → Yellow `\033[38;2;255;215;0m`
- Clean/OK → Green `\033[38;2;80;250;123m`
- Label/dim → `\033[38;2;100;170;255m` / `\033[38;2;128;128;128m`

**Done when:** Every state the script can be in has a corresponding output line designed.

---

### Step 3: Implement the script

Create `statusline.sh` in the project root. The structure depends on the execution
model chosen in Step 1 (based on the benchmark results).

**If using inline execution** (operation benchmarked at <300ms):
1. **Configuration block** — env vars with defaults (`TOOL_BIN`, display toggles)
2. **Color constants** — define all ANSI codes as variables; never inline raw escapes
3. **Read stdin** — `SESSION=$(cat)` — captures Claude's JSON; parse fields as needed
4. **Run the operation directly** — execute the command and capture its output/exit code
5. **Build segment functions** — one per data source; handle unavailable/auth error states
6. **Compose final output** — `printf` the assembled line with separators

**If using background + cache** (operation benchmarked at >300ms or variable):
1. **Configuration block** — env vars with defaults (`TOOL_BIN`, `CACHE_TTL`, display toggles)
2. **Color constants** — define all ANSI codes as variables; never inline raw escapes
3. **Read stdin** — `SESSION=$(cat)` — captures Claude's JSON; parse fields as needed
4. **Determine project root** — `git rev-parse --show-toplevel 2>/dev/null || pwd`
5. **Cache paths** — hash the project path with `cksum` for stable, per-project cache keys
6. **`file_age()` helper** — returns age in seconds of any file; 999999 if missing
7. **`trigger_scan_bg()` function** — atomic `mkdir`-based lock, runs scan in subshell, captures exit code explicitly, writes noscan sentinel on exit 3
8. **Trigger stale scans** — compare age to TTL; call trigger functions
9. **Build segment functions** — one per data source; check cache/noscan/lock states and return the formatted string for that segment
10. **Compose final output** — `printf` the assembled line with separators

See `references/implementation-patterns.md` for full patterns for both models.

Critical rules:
- Use `|| exit_code=$?` to capture exit codes — never `|| true`, which loses them
- Some CLI tools use exit code 3 to mean "no supported project/files" (distinct from a scan error) — detect this and write a `.noscan` sentinel rather than retrying indefinitely
- Use `mkdir` for locking (POSIX-atomic), not `flock` or file touches
- Make a single `jq` call per cache file — multiple calls on the same file are slow
- The script runs in Claude's cwd — use `pwd` / `git rev-parse` to find the project

Make the script executable: `chmod +x statusline.sh`

**Configuration env vars — naming convention:**
Every user-tunable knob should be an env var with a sane default. Use a consistent naming prefix so vars are discoverable:

```bash
# In statusline.sh — configuration block at the top
TOOL_BIN="${TOOL_BIN:-tool}"                   # path to the CLI binary
TOOL_STATUSLINE_TTL="${TOOL_STATUSLINE_TTL:-300}"  # cache TTL in seconds
TOOL_SHOW_LOW="${TOOL_SHOW_LOW:-false}"        # boolean display toggle
TOOL_SCAN_ARGS="${TOOL_SCAN_ARGS:-}"           # extra args forwarded to scan command
```

Users can set these in their shell profile or in `~/.claude/settings.json` under `"env"` — document both paths. The `TOOL_BIN` var is especially important on macOS where tools installed via `nvm`, `fnm`, `rbenv`, or `pyenv` are only in PATH in interactive shells; background subshells run by the statusline may not inherit that PATH. Always document the `TOOL_BIN` override and recommend pinning it if the tool is installed via a version manager:
```bash
export TOOL_BIN=$(which tool)   # resolve once in your shell profile
```

**Done when:** The script runs without errors from `echo '{}' | ./statusline.sh` in the project directory.

---

### Step 4: Install into Claude Code

Add the statusline to `~/.claude/settings.json`. The value must be an **object**, not a string — a plain string path will cause a settings error on startup:

```json
{
  "statusLine": {
    "type": "command",
    "command": "/absolute/path/to/statusline.sh"
  }
}
```

Use `jq` to update non-destructively (preserves all other settings):
```bash
jq --arg p "/absolute/path/to/statusline.sh" \
   '.statusLine = {"type": "command", "command": $p}' \
   ~/.claude/settings.json > ~/.claude/settings.json.tmp \
   && mv ~/.claude/settings.json.tmp ~/.claude/settings.json
```

**Done when:** `cat ~/.claude/settings.json` shows the statusLine object with the correct absolute path.

---

### Step 5: Create install.sh

**Always create an `install.sh`** in the project root. This is not optional — every statusline needs one so users can install and uninstall it without manually editing JSON. Use `assets/install.sh.template` as the starting point and fill in the placeholders.

A correct installer must:
1. Use **colored output** — define `R`/`GREEN`/`YELLOW`/`RED`/`BOLD` ANSI variables and `ok()`/`info()`/`die()` helpers that use them; use `printf` not `echo`; print a bold tool name header at the start
2. Check that required CLI tools are installed (with install hints if missing)
3. Run an **auth preflight** — verify the tool is authenticated *before* touching settings
4. Update `~/.claude/settings.json` non-destructively using `jq`
5. Support `--remove` to cleanly uninstall
6. Print the env vars users can set to configure behavior

**Auth preflight — important distinction:** there are two different commands for every authenticated tool:
- A **check** command (e.g. `tool whoami`, `gh auth status`) — safe to run anytime; exit 0 = authenticated
- A **do-auth** command (e.g. `tool auth`, `gh auth login`) — only run this if not authenticated; on some tools re-running when already authed resets or replaces credentials

The installer should run the *check* command and only instruct the user to run the *do-auth* command if the check fails. Never silently run the auth command unconditionally.

Make it executable: `chmod +x install.sh`

**Done when:** `./install.sh` runs cleanly with colored output and `./install.sh --remove` cleanly removes the statusLine entry.

---

### Step 6: Test every state

Do not rely on the live statusline alone — seed the cache manually to exercise every state:

```bash
CACHE_DIR="$HOME/.cache/<tool>-statusline"
HASH=$(printf '%s' "$(git rev-parse --show-toplevel)" | cksum | cut -d' ' -f1)

# Test: clean/no issues
echo '<valid empty result JSON>' > "$CACHE_DIR/${HASH}.json"
echo '{}' | ./statusline.sh

# Test: issues found
echo '<valid result JSON with findings>' > "$CACHE_DIR/${HASH}.json"
echo '{}' | ./statusline.sh

# Test: no supported project
printf '3' > "$CACHE_DIR/${HASH}.noscan"; rm -f "$CACHE_DIR/${HASH}.json"
echo '{}' | ./statusline.sh

# Test: scanning in progress
mkdir "$CACHE_DIR/${HASH}.lock" 2>/dev/null
rm -f "$CACHE_DIR/${HASH}.json" "$CACHE_DIR/${HASH}.noscan"
echo '{}' | ./statusline.sh
rmdir "$CACHE_DIR/${HASH}.lock"

# Test: auth error
echo 'Authentication failed' > "$CACHE_DIR/${HASH}.err"
echo '{}' | ./statusline.sh
```

Check the raw output for correct ANSI escape codes:
```bash
echo '{}' | ./statusline.sh | cat -v
```
`^[[38;2;...m` in the output means ANSI is working correctly.

Repeat testing in the project directory where the data source is valid, and in a directory where no supported files exist, to confirm the noscan path works.

**Done when:** All designed states produce the expected output line with correct colors and content.

---

### Step 7: Write documentation

Create a `README.md` covering:
1. **What it shows** — annotated output line explaining every token
2. **Why it's useful** — the problem it solves for a developer, not a feature list
3. **What scans/data sources run** — and why those specifically
4. **Prerequisites** — tools required, accounts, authentication
5. **Quick start** — clone → auth check → `./install.sh` (three steps max)
6. **Manual installation** — direct `settings.json` snippet for users who don't want the installer
7. **Uninstall** — `./install.sh --remove` and how to clear the cache
8. **Configuration** — table of env vars with defaults, and example of setting them via `settings.json` `"env"` block
9. **Troubleshooting** — how to read the error log, force a cache refresh, fix auth; **include the binary PATH pinning note** for tools installed via version managers

See `assets/README.md.template` for a ready-to-fill skeleton with all sections pre-structured.

Annotate the output line token by token — users shouldn't have to guess what `H:4 M:2 (6↑)` means:

```
🔒 snyk │ deps H:4 M:2 (6↑) │ code H:2 M:3 │ test-project · 5m ago ⟳
          ^^^^ ^^^  ^^^  ^^^^   ^^^^ ^^^  ^^^   ^^^^^^^^^^^^^ ^^^^^^^^ ^
          [A]  [B]  [C]  [D]   [E]  [F]  [G]       [H]          [I]   [J]
```

[A] scan type label, [B][C] severity counts, [D] fixable, [E–G] same for second scan,
[H] project name, [I] cache age, [J] background scan active

**Done when:** A developer unfamiliar with the project can install and use the statusline using the README alone.

---

## Examples

**Example 1: Building a security statusline from scratch**

User says: "I want to see Snyk vulnerability counts in my Claude statusline."

Actions:
1. Confirm data source: `snyk test --json` (SCA) and `snyk code test --json` (SAST)
2. Run both commands in the test project to capture real output; note JSON structures,
   exit codes (0=clean, 1=vulns, 3=no manifest), and scan duration
3. Design two segments: `deps H:N M:N (N↑)` and `code H:N M:N`, plus `· Xs ago ⟳`
4. Implement `statusline.sh` with two background scan functions and separate caches;
   parse SCA with `.vulnerabilities[]` and SAST with `.runs[0].results[]` (SARIF)
5. Install into `~/.claude/settings.json` using the `{type, command}` object format
6. Test all states by seeding cache files; verify ANSI codes with `cat -v`
7. Write README with annotated output line and troubleshooting section

Result: A single status bar line showing live dep and code security status, refreshing
every 5 minutes in the background, with zero impact on Claude's response time.

---

**Example 2: Adding a second data source to an existing statusline**

User says: "My statusline shows git branch — can I also add the last CI run status?"

Actions:
1. Read the existing `statusline.sh` to understand its cache and output structure
2. Identify the CI data source (e.g. `gh run list --limit 1 --json status,conclusion`)
3. Add new cache variables (`CI_CACHE`, `CI_LOCK`, `CI_NOSCAN`) following the existing pattern
4. Add a `trigger_ci_bg()` function using the same atomic-lock pattern
5. Add a `build_ci_segment()` function; handle the case where `gh` is not authenticated
6. Compose the new segment into the existing `printf` output line
7. Test: seed CI cache with a passing result, a failing result, and no GH auth

Result: The statusline now shows both git state and CI status on one line, each
refreshing independently, without either scan blocking the other.

---

**Example 3: Debugging a statusline stuck on "scanning..."**

User says: "My statusline has been saying 'scanning...' for 20 minutes."

Actions:
1. Check whether the lock directory is still present: `ls ~/.cache/<tool>-statusline/*.lock`
2. If the lock exists but no background process is running, it's a stale lock from a
   crashed scan — remove it: `rmdir ~/.cache/<tool>-statusline/*.lock`
3. Run the scan command manually in the project dir to check for errors:
   `snyk test --json 2>&1 | head -20`
4. Check the error log: `cat ~/.cache/<tool>-statusline/*.err`
5. Common root causes: auth expired (re-run `snyk auth`), no manifest in directory
   (statusline should show "no manifest" not "scanning"), background process killed
6. Force a fresh scan by deleting the cache: `rm -rf ~/.cache/<tool>-statusline/`
7. Re-run `echo '{}' | ./statusline.sh` and watch for the "initializing..." → "scanning..." → result progression

Result: Root cause identified and resolved; statusline shows current data.

---

## Troubleshooting

**Error:** `Settings Error: statusLine: Expected object, but received string`
Cause: `settings.json` has `"statusLine": "/path/to/script"` as a plain string.
Solution: Change it to the object format: `"statusLine": {"type": "command", "command": "/path/to/script"}`. Run the `jq` update command from Step 4.

---

**Error:** Statusline stuck on "scanning..." indefinitely.
Cause: A stale lock directory left by a crashed background scan process.
Solution: Delete the lock: `rmdir ~/.cache/<tool>-statusline/*.lock`. The next render will trigger a fresh scan.

---

**Error:** Statusline shows "initializing..." after every Claude restart.
Cause: The cache TTL has expired between sessions, or the cache directory is being
cleared on restart.
Solution: Increase `CACHE_TTL` (e.g. `export TOOL_STATUSLINE_TTL=3600`), or check
whether something is clearing `~/.cache` on startup.

---

**Error:** Colors don't render — raw escape codes like `^[[38;2;...` show in the bar.
Cause: The terminal or Claude Code version doesn't support RGB ANSI. Less likely:
escape codes are being double-escaped.
Solution: Verify codes are defined as `$'...'` literals (e.g. `RED=$'\033[38;2;255;85;85m'`),
not as regular strings with `\033`. Use `cat -v` to inspect raw output.

---

**Error:** Background scan never completes — cache file never appears.
Cause: The scan command is failing silently. The `|| true` pattern suppresses the
exit code, so errors go undetected.
Solution: Run the scan command manually. Check the `.err` file in the cache dir.
Ensure exit code 3 ("no supported project") writes a `.noscan` sentinel instead of
being treated as a generic failure — otherwise the script retries every render.
