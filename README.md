# statusline-creator

[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Skill Version](https://img.shields.io/badge/version-1.0.0-brightgreen)](./skills/statusline-creator/SKILL.md)
[![Requires: Claude Code](https://img.shields.io/badge/requires-Claude%20Code-orange)](https://claude.ai/code)
[![Shell: bash](https://img.shields.io/badge/shell-bash-lightgrey)](https://www.gnu.org/software/bash/)
[![Dependency: jq](https://img.shields.io/badge/dep-jq-lightgrey)](https://stedolan.github.io/jq/)

A Claude Code agent skill that builds and extends **statuslines** — live status bars that display real-time data directly inside your Claude Code session after every response.

Example of a Claude Code statusline built for the Snyk security scanner:

```
🔒 snyk │ deps H:4 M:2 (6↑) │ code H:2 M:3 │ my-project · 5m ago ⟳
```

---

## What is a statusline?

A Claude Code statusline is a shell script that runs after every assistant response. It receives session metadata on stdin and prints a colored line to stdout — giving you live, contextual data without leaving the session.

Common use cases:
- **Security** — Snyk vulnerability counts, Trivy findings, Semgrep results
- **Git/project state** — branch, dirty files, CI status, PR state
- **API usage** — context window %, session cost, rate limit headroom
- **Build/test state** — last test result, coverage, lint status
- **Cloud/infra** — Kubernetes pod state, deploy status, environment

The key design constraint: the statusline must render **instantly**. Slow operations (network calls, CLI scans) run in the background and write to a local cache. The script always renders from cache — never from a live call.

---

## What this skill does

The `statusline-creator` skill drives the entire workflow from idea to a working, installed, tested statusline:

1. **Understands your data source** — asks what you want to show, inspects the CLI command and its output format, establishes all display states
2. **Designs the output line** — plans every state the bar can be in (scanning, clean, findings, auth error, no supported files) before writing a line of code
3. **Implements the script** — creates `statusline.sh` following a production-ready pattern: atomic locking, background scans, per-project cache keys, single-`jq`-call parsing, ANSI RGB colors
4. **Installs into Claude Code** — generates an `install.sh` (with auth preflight and `--remove` support) and updates `~/.claude/settings.json` non-destructively using `jq`
5. **Tests every state** — seeds the cache manually to verify each display state without waiting for a live scan
6. **Writes documentation** — produces a README covering annotated output, configuration env vars, and troubleshooting (including the binary PATH pinning issue for version managers)

---

## Installation

### Prerequisites

- [Claude Code](https://claude.ai/code) installed and authenticated
- `bash` and `jq` available in your shell
- Any external tools your statusline reads from (e.g. `snyk`, `gh`, `kubectl`) installed separately

### Add the skill to Claude Code

Copy the skill file into your Claude Code skills directory:

```bash
# Create the skills directory if it doesn't exist
mkdir -p ~/.claude/skills

# Copy the skill
cp skills/statusline-creator/SKILL.md ~/.claude/skills/statusline-creator.md
```

Or clone this repo and symlink it:

```bash
git clone https://github.com/lirantal/statusline-creator-skill
ln -s "$(pwd)/statusline-creator-skill/skills/statusline-creator/SKILL.md" \
      ~/.claude/skills/statusline-creator.md
```

### Verify installation

Start a new Claude Code session and ask:

```
Create a statusline that shows my git branch and context window usage.
```

The skill activates automatically when you describe wanting live data in your session — you don't need to prefix anything or name the skill explicitly.

---

## Usage

Just describe what you want to see. The skill picks up on natural-language requests:

| What you say | What the skill builds |
|---|---|
| `"Create a statusline"` | Asks what data you want, then builds from scratch |
| `"Make a statusline for Snyk"` | Security statusline with dep + code scan segments |
| `"Add CI status to my statusline"` | Extends an existing `statusline.sh` with a new segment |
| `"Show my context window in the status bar"` | Inline statusline reading the stdin JSON |
| `"I want live feedback during my Claude session"` | Prompts for specifics, then builds |

### Example session

```
You: I want to see Snyk vulnerability counts in my Claude statusline.

Claude: [runs snyk test --json to inspect output, designs display states,
         writes statusline.sh, installs it, tests all states, writes README]

Result:
🔒 snyk │ deps C:0 H:4 M:2 (6↑) │ code H:2 M:3 │ my-app · 3m ago
```

---

## Skill reference

The full skill source lives at [`skills/statusline-creator/SKILL.md`](./skills/statusline-creator/SKILL.md).

Supporting reference files:

| File | Purpose |
|---|---|
| [`references/protocol.md`](./skills/statusline-creator/references/protocol.md) | Stdin JSON schema, `settings.json` format, ANSI output rules |
| [`references/implementation-patterns.md`](./skills/statusline-creator/references/implementation-patterns.md) | Caching pattern, locking, state machine, `jq` recipes, env var conventions, binary path pinning, auth check pattern |
| [`assets/install.sh.template`](./skills/statusline-creator/assets/install.sh.template) | Generic installer template — dependency checks, auth preflight, settings update, `--remove` mode |
| [`assets/README.md.template`](./skills/statusline-creator/assets/README.md.template) | README skeleton with all standard sections pre-structured |

---

## How the generated statusline works

The skill produces a `statusline.sh` that follows this structure:

```
[configuration]   env vars with defaults (binary path, cache TTL, toggles)
[color constants] ANSI RGB codes as $'...' bash literals
[read stdin]      SESSION=$(cat) — consumes Claude's JSON
[project root]    git rev-parse --show-toplevel || pwd
[cache paths]     per-project keys hashed with cksum
[file_age()]      returns age in seconds; 999999 if missing
[trigger_*_bg()]  atomic mkdir lock → background scan → write cache
[build_*()]       check state order: scanning → noscan → auth_err → initializing → result
[printf output]   assemble segments with │ separators and ANSI colors
```

State machine for each scan slot (checked in order):

```
1. SCANNING      lock dir exists, no cache    →  "scanning..."
2. NOSCAN        sentinel exists, no cache    →  "no supported files"
3. AUTH_ERR      err file has auth keywords   →  "⚠ auth required"
4. INITIALIZING  nothing exists yet           →  "initializing..."
5. HAS_RESULT    cache file exists            →  parsed result + optional ⟳ spinner
```

---

## Troubleshooting

**`Settings Error: statusLine: Expected object, but received string`**
The `settings.json` value must be an object, not a plain path string. Fix:
```bash
jq --arg p "/absolute/path/to/statusline.sh" \
   '.statusLine = {"type": "command", "command": $p}' \
   ~/.claude/settings.json > ~/.claude/settings.json.tmp \
   && mv ~/.claude/settings.json.tmp ~/.claude/settings.json
```

**Statusline stuck on "scanning..." indefinitely**
A stale lock directory from a crashed background process. Remove it:
```bash
rmdir ~/.cache/<tool>-statusline/*.lock
```

**Colors show as raw escape codes (`\033[38;2;...`)**
Colors must be defined as `$'...'` bash literals, not regular strings. Inspect with:
```bash
echo '{}' | ./statusline.sh | cat -v
# Correct: ^[[38;2;255;85;85m
# Wrong:   \033[38;2;255;85;85m
```

**Background scan never writes a cache file**
Run the scan command manually in the project directory and check the `.err` file:
```bash
cat ~/.cache/<tool>-statusline/*.err
```

---

## License

[Apache 2.0](./LICENSE) — see the LICENSE file for details.

---

## Author

**Liran Tal** — [@lirantal](https://github.com/lirantal)
