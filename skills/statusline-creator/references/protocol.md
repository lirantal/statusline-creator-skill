# Claude Code Statusline Protocol

**Authoritative reference:** https://code.claude.com/docs/en/statusline.md

---

## How it works

Claude Code invokes the statusline script after every assistant message. The script:
1. Receives a JSON object on **stdin** with session metadata
2. Runs any computation it needs (always from cache — see `implementation-patterns.md`)
3. Prints one or more lines of text to **stdout**
4. Exits 0

The output is rendered in Claude Code's status bar. The script runs in the **same
working directory** as the Claude session — this is how it knows which project to act on.

Updates are debounced at 300ms. If a new update triggers while the script is running,
the in-flight execution is cancelled. Scripts that exit non-zero or produce no output
cause the status bar to go blank.

---

## settings.json format

Register the statusline in `~/.claude/settings.json` (user settings) or the project's
`.claude/settings.json`:

```json
{
  "statusLine": {
    "type": "command",
    "command": "/absolute/path/to/statusline.sh",
    "padding": 2
  }
}
```

**`command`** — path to a script file, or an inline shell command:
```json
"command": "jq -r '.model.display_name'"
```

**`padding`** — optional integer, adds extra horizontal spacing in characters (default: `0`).

**The value must be an object.** A plain string path causes a startup error:
```
Settings Error: statusLine: Expected object, but received string
```

Update non-destructively with `jq`:
```bash
jq --arg p "/absolute/path/to/statusline.sh" \
   '.statusLine = {"type": "command", "command": $p}' \
   ~/.claude/settings.json > ~/.claude/settings.json.tmp \
   && mv ~/.claude/settings.json.tmp ~/.claude/settings.json
```

The `/statusline` Claude Code command can also generate and install a script for you
from a natural language description — useful for getting a working draft quickly.

---

## Session JSON (stdin)

Full schema passed on stdin on every invocation:

```json
{
  "cwd": "/current/working/directory",
  "session_id": "abc123...",
  "transcript_path": "/path/to/transcript.jsonl",
  "model": {
    "id": "claude-opus-4-6",
    "display_name": "Opus"
  },
  "workspace": {
    "current_dir": "/current/working/directory",
    "project_dir": "/original/project/directory"
  },
  "version": "1.0.80",
  "output_style": { "name": "default" },
  "cost": {
    "total_cost_usd": 0.01234,
    "total_duration_ms": 45000,
    "total_api_duration_ms": 2300,
    "total_lines_added": 156,
    "total_lines_removed": 23
  },
  "context_window": {
    "total_input_tokens": 15234,
    "total_output_tokens": 4521,
    "context_window_size": 200000,
    "used_percentage": 8,
    "remaining_percentage": 92,
    "current_usage": {
      "input_tokens": 8500,
      "output_tokens": 1200,
      "cache_creation_input_tokens": 5000,
      "cache_read_input_tokens": 2000
    }
  },
  "exceeds_200k_tokens": false,
  "rate_limits": {
    "five_hour": { "used_percentage": 23.5, "resets_at": 1738425600 },
    "seven_day": { "used_percentage": 41.2, "resets_at": 1738857600 }
  },
  "vim": { "mode": "NORMAL" },
  "agent": { "name": "my-agent" },
  "worktree": {
    "name": "my-feature",
    "path": "/path/to/.claude/worktrees/my-feature",
    "branch": "worktree-my-feature",
    "original_cwd": "/path/to/project",
    "original_branch": "main"
  }
}
```

### Field reference

| Field | Type | Notes |
|---|---|---|
| `cwd` | string | Current working directory |
| `workspace.current_dir` | string | Same as `cwd`; preferred |
| `workspace.project_dir` | string | Directory where Claude was launched; may differ from `cwd` |
| `model.id` | string | Model identifier, e.g. `claude-opus-4-6` |
| `model.display_name` | string | Short display name, e.g. `Opus` |
| `cost.total_cost_usd` | number | Cumulative session cost in USD |
| `cost.total_duration_ms` | number | Wall-clock ms since session started |
| `cost.total_api_duration_ms` | number | ms spent waiting for API responses |
| `cost.total_lines_added/removed` | number | Lines of code changed this session |
| `context_window.used_percentage` | number | Pre-calculated % of context used; may be null early in session |
| `context_window.remaining_percentage` | number | Pre-calculated % remaining |
| `context_window.context_window_size` | number | Max tokens (200k or 1M for extended) |
| `context_window.current_usage` | object\|null | Token breakdown from last API call; null before first call |
| `exceeds_200k_tokens` | boolean | Whether last response exceeded 200k total tokens |
| `rate_limits.five_hour.used_percentage` | number | % of 5-hour rolling limit used |
| `rate_limits.seven_day.used_percentage` | number | % of 7-day limit used |
| `rate_limits.*.resets_at` | number | Unix epoch seconds when window resets |
| `session_id` | string | Unique session ID |
| `version` | string | Claude Code version |
| `vim.mode` | string | `NORMAL` or `INSERT`; absent unless vim mode enabled |
| `agent.name` | string | Present only with `--agent` flag or agent settings |
| `worktree.name/path/branch` | string | Present only in `--worktree` sessions |

### Conditional fields

These fields are absent (not null) when not applicable — handle with `// empty` or `// "default"` in jq:

- `vim` — only present when vim mode is enabled
- `agent` — only present with `--agent` flag or agent settings
- `worktree` — only present in `--worktree` sessions
- `rate_limits` — only present for Claude.ai Pro/Max subscribers after the first API response
- `context_window.current_usage` — `null` before the first API call

---

## Output format

The script prints to stdout. Claude Code renders:
- **ANSI escape codes** — including 256-color and RGB (`\033[38;2;R;G;Bm`)
- **Unicode** — emoji, box-drawing characters, symbols
- **OSC 8 hyperlinks** — `\033]8;;URL\atext\033]8;;\a` for clickable links
- **Multiple lines** — each `echo` creates a separate row in the status area

Status bar width is limited — keep output concise to avoid truncation.

---

## Script requirements

- Must be executable: `chmod +x statusline.sh`
- Must consume stdin even if ignoring it: `SESSION=$(cat)`
- Must exit 0 — non-zero exit clears the status bar
- Must complete quickly — slow scripts block updates; slow operations must be cached

---

## Testing

```bash
# Basic test
echo '{"model":{"display_name":"Opus"},"context_window":{"used_percentage":25},"cwd":"/my/project"}' \
  | ./statusline.sh

# Inspect raw ANSI codes
echo '{}' | ./statusline.sh | cat -v
# Correct output contains: ^[[38;2;...m (ESC code)
# Wrong output contains:   \033[38;2;...m (literal backslash)
```
