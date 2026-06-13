# safe-scripts Plugin Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a Claude Code plugin with three hooks (SessionStart, PreToolUse, PermissionRequest) and a behavior-shaping skill that intercepts Bash tool calls, redirects to pre-approved safe scripts, and offers to generalize new commands into reusable scripts to eliminate permission dialogs.

**Architecture:** SessionStart injects the script catalog at session start so Claude proactively prefers safe scripts. PreToolUse intercepts any Bash call matching the manifest (regex match for standard commands, structural + semantic two-tier match for heredocs) and blocks with a redirect instruction. PermissionRequest injects a hint so Claude offers to save after the first approval. A behavior-shaping skill encodes all four behaviors (proactive preference, redirect on match, offer on permission, save procedure). Scripts and manifest live at `~/.claude/safe-scripts/` by default, overridable via `.claude/safe-scripts-config.json`.

**Tech Stack:** Bash (hooks + lib), jq ≥ 1.5 (JSON parsing), SKILL.md (Claude behavior shaping)

---

## File Map

| File | Responsibility |
|---|---|
| `.claude-plugin/plugin.json` | Plugin identity and metadata |
| `package.json` | Version and entry point |
| `hooks/hooks.json` | Hook event bindings |
| `hooks/run-hook.cmd` | Cross-platform hook runner (Windows compat) |
| `hooks/session-start` | SessionStart: inject catalog + behavioral rules |
| `hooks/pre-tool-use` | PreToolUse: manifest match → block + redirect |
| `hooks/permission-request` | PermissionRequest: inject [OFFER_SAFE_SCRIPT] hint |
| `scripts/lib.sh` | Shared: config resolution, manifest load, pattern match, JSON emit |
| `skills/safe-scripts/SKILL.md` | Claude behavior: proactive preference, save procedure, generalization rules |
| `tests/test-lib.sh` | Unit tests for lib.sh |
| `tests/test-session-start.sh` | Tests for session-start hook |
| `tests/test-pre-tool-use.sh` | Tests for pre-tool-use hook (standard + heredoc) |
| `tests/test-permission-request.sh` | Tests for permission-request hook |
| `tests/run-tests.sh` | Test runner that executes all test files |
| `README.md` | Installation, usage, configuration |
| `LICENSE` | MIT |

---

### Task 1: Scaffold — plugin manifest, hooks.json, run-hook.cmd

**Files:**
- Create: `.claude-plugin/plugin.json`
- Create: `package.json`
- Create: `hooks/hooks.json`
- Create: `hooks/run-hook.cmd`

- [ ] **Step 1: Create plugin manifest**

```bash
mkdir -p .claude-plugin hooks scripts skills/safe-scripts tests
```

Write `.claude-plugin/plugin.json`:
```json
{
  "name": "safe-scripts",
  "description": "Build a library of pre-approved safe scripts to eliminate repetitive permission dialogs",
  "version": "1.0.0",
  "author": {
    "name": "Your Name"
  },
  "license": "MIT",
  "keywords": ["permissions", "safety", "scripts", "automation", "bash"]
}
```

- [ ] **Step 2: Create package.json**

```json
{
  "name": "safe-scripts",
  "version": "1.0.0",
  "description": "Claude Code plugin: pre-approved safe scripts to eliminate permission dialogs"
}
```

- [ ] **Step 3: Create hooks/hooks.json**

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|clear|compact",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd\" session-start",
            "async": false
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd\" pre-tool-use"
          }
        ]
      }
    ],
    "PermissionRequest": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "\"${CLAUDE_PLUGIN_ROOT}/hooks/run-hook.cmd\" permission-request"
          }
        ]
      }
    ]
  }
}
```

- [ ] **Step 4: Create hooks/run-hook.cmd** (cross-platform polyglot — handles Windows bash lookup)

```
: << 'CMDBLOCK'
@echo off
if "%~1"=="" (
    echo run-hook.cmd: missing script name >&2
    exit /b 1
)
set "HOOK_DIR=%~dp0"
if exist "C:\Program Files\Git\bin\bash.exe" (
    "C:\Program Files\Git\bin\bash.exe" "%HOOK_DIR%%~1" %2 %3 %4 %5
    exit /b %ERRORLEVEL%
)
if exist "C:\Program Files (x86)\Git\bin\bash.exe" (
    "C:\Program Files (x86)\Git\bin\bash.exe" "%HOOK_DIR%%~1" %2 %3 %4 %5
    exit /b %ERRORLEVEL%
)
where bash >nul 2>nul
if %ERRORLEVEL% equ 0 (
    bash "%HOOK_DIR%%~1" %2 %3 %4 %5
    exit /b %ERRORLEVEL%
)
exit /b 0
CMDBLOCK

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
exec bash "${SCRIPT_DIR}/${1}" "${@:2}"
```

- [ ] **Step 5: Verify structure**

Run: `find . -not -path './.git/*' | sort`

Expected output includes:
```
./.claude-plugin/plugin.json
./hooks/hooks.json
./hooks/run-hook.cmd
./scripts
./skills/safe-scripts
./tests
```

- [ ] **Step 6: Commit**

```bash
git add .claude-plugin/ hooks/ scripts/ skills/ tests/ package.json
git commit -m "feat: scaffold plugin structure — manifest, hooks.json, run-hook.cmd"
```

---

### Task 2: Shared library — scripts/lib.sh

**Files:**
- Create: `scripts/lib.sh`
- Create: `tests/test-lib.sh`

- [ ] **Step 1: Write failing tests**

Write `tests/test-lib.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
PASS=0; FAIL=0
SCRIPT_DIR="$(cd "$(dirname "$0")/.." && pwd)"
source "${SCRIPT_DIR}/scripts/lib.sh"

assert_eq() {
    if [ "$1" = "$2" ]; then PASS=$((PASS+1)); echo "PASS: $3"
    else FAIL=$((FAIL+1)); echo "FAIL: $3"; echo "  want: $2"; echo "  got:  $1"; fi
}
assert_true() {
    if "$@" 2>/dev/null; then PASS=$((PASS+1)); echo "PASS: $*"
    else FAIL=$((FAIL+1)); echo "FAIL: $*"; fi
}
assert_false() {
    if ! "$@" 2>/dev/null; then PASS=$((PASS+1)); echo "PASS: (not) $*"
    else FAIL=$((FAIL+1)); echo "FAIL: expected false: $*"; fi
}

TMPDIR_TEST=$(mktemp -d)
trap "rm -rf '$TMPDIR_TEST'" EXIT

# --- get_safe_scripts_dir ---
assert_eq "$(SAFE_SCRIPTS_DIR="$TMPDIR_TEST" get_safe_scripts_dir)" "$TMPDIR_TEST" \
    "get_safe_scripts_dir: env override"

assert_eq "$(get_safe_scripts_dir)" "${HOME}/.claude/safe-scripts" \
    "get_safe_scripts_dir: default path"

mkdir -p "$TMPDIR_TEST/project/.claude"
printf '{"safe_scripts_dir":"%s/custom"}' "$TMPDIR_TEST" > "$TMPDIR_TEST/project/.claude/safe-scripts-config.json"
assert_eq "$(cd "$TMPDIR_TEST/project" && get_safe_scripts_dir)" "$TMPDIR_TEST/custom" \
    "get_safe_scripts_dir: project config override"

# --- load_manifest ---
assert_eq "$(load_manifest "$TMPDIR_TEST/empty")" '{"version":1,"scripts":[]}' \
    "load_manifest: missing dir returns empty manifest"

mkdir -p "$TMPDIR_TEST/with-manifest"
printf '{"version":1,"scripts":[{"name":"test-script"}]}' > "$TMPDIR_TEST/with-manifest/manifest.json"
assert_eq "$(load_manifest "$TMPDIR_TEST/with-manifest" | jq -r '.scripts[0].name')" "test-script" \
    "load_manifest: reads manifest file"

# --- is_heredoc ---
assert_true is_heredoc "python3 << 'EOF'"
assert_true is_heredoc 'bash <<EOF'
assert_true is_heredoc "node << 'SCRIPT'"
assert_false is_heredoc "git log --oneline -10 -- src/App.tsx"
assert_false is_heredoc "python3 analyze.py --input data.csv"

# --- find_matching_script ---
MANIFEST='{"version":1,"scripts":[{"name":"git-file-log","description":"Show git history","script":"git-file-log.sh","usage":"git-file-log <file> [--limit N]","patterns":["^git log (--oneline )?(-[0-9]+ )?-- .+"]}]}'

MATCH=$(find_matching_script "git log --oneline -10 -- src/Button.tsx" "$MANIFEST")
assert_eq "$(echo "$MATCH" | jq -r '.name')" "git-file-log" \
    "find_matching_script: matches git log variant"

MATCH=$(find_matching_script "git log -- src/App.tsx" "$MANIFEST")
assert_eq "$(echo "$MATCH" | jq -r '.name')" "git-file-log" \
    "find_matching_script: matches git log without --oneline"

NO_MATCH=$(find_matching_script "ls -la" "$MANIFEST")
assert_eq "$NO_MATCH" "" "find_matching_script: no match returns empty"

# --- find_heredoc_candidates ---
HDMANIFEST='{"version":1,"scripts":[{"name":"analyze-csv","heredoc":true,"description":"Analyze a CSV"},{"name":"git-log","description":"Git log"}]}'
CANDIDATES=$(find_heredoc_candidates "$HDMANIFEST")
assert_true echo "$CANDIDATES" | grep -q "analyze-csv"
assert_false echo "$CANDIDATES" | grep -q "git-log"

echo ""
echo "Results: ${PASS} passed, ${FAIL} failed"
[ "$FAIL" -eq 0 ]
```

- [ ] **Step 2: Run tests — verify they fail**

```bash
bash tests/test-lib.sh
```

Expected: errors about `get_safe_scripts_dir: command not found` (or similar — lib.sh doesn't exist yet)

- [ ] **Step 3: Write scripts/lib.sh**

```bash
#!/usr/bin/env bash
# Shared utilities for safe-scripts hooks.
# Source this file — do not execute directly.

# Resolve the safe-scripts directory.
# Priority: SAFE_SCRIPTS_DIR env (tests/override) > .claude/safe-scripts-config.json > default.
get_safe_scripts_dir() {
    if [ -n "${SAFE_SCRIPTS_DIR:-}" ]; then
        echo "$SAFE_SCRIPTS_DIR"
        return
    fi
    local config=".claude/safe-scripts-config.json"
    if [ -f "$config" ]; then
        local dir
        dir=$(jq -r '.safe_scripts_dir // empty' "$config" 2>/dev/null)
        if [ -n "$dir" ]; then
            # Resolve relative paths against cwd
            if [[ "$dir" != /* ]]; then
                dir="$(pwd)/${dir}"
            fi
            echo "$dir"
            return
        fi
    fi
    echo "${HOME}/.claude/safe-scripts"
}

# Read manifest.json from dir, or return an empty-scripts manifest.
load_manifest() {
    local dir="$1"
    local manifest="${dir}/manifest.json"
    if [ -f "$manifest" ]; then
        cat "$manifest"
    else
        printf '{"version":1,"scripts":[]}'
    fi
}

# Return true if the command is a heredoc inline script.
is_heredoc() {
    local command="$1"
    echo "$command" | grep -qE '<<\s*'"'"'?[A-Z_a-z]+'
}

# Find the first manifest entry whose patterns match the command.
# Returns the JSON object of the matching entry, or empty string.
# Skips entries with heredoc:true.
find_matching_script() {
    local command="$1"
    local manifest="$2"
    echo "$manifest" | jq -c --arg cmd "$command" '
        [ .scripts[] | select(
            (.heredoc // false | not) and
            (.patterns // [] | map(
                . as $pat | try ($cmd | test($pat)) catch false
            ) | any)
        ) ] | first // empty
    ' 2>/dev/null
}

# Return names+descriptions of scripts flagged heredoc:true.
find_heredoc_candidates() {
    local manifest="$1"
    echo "$manifest" | jq -r \
        '.scripts[] | select(.heredoc == true) | "- " + .name + ": " + .description' \
        2>/dev/null
}

# Emit platform-aware additionalContext JSON.
# Usage: emit_context <context_string> <event_name>
emit_context() {
    local context="$1"
    local event_name="$2"
    if [ -n "${CURSOR_PLUGIN_ROOT:-}" ]; then
        jq -n --arg ctx "$context" '{"additional_context":$ctx}'
    elif [ -n "${CLAUDE_PLUGIN_ROOT:-}" ] && [ -z "${COPILOT_CLI:-}" ]; then
        jq -n --arg ctx "$context" --arg evt "$event_name" \
            '{"hookSpecificOutput":{"hookEventName":$evt,"additionalContext":$ctx}}'
    else
        jq -n --arg ctx "$context" '{"additionalContext":$ctx}'
    fi
}
```

- [ ] **Step 4: Run tests — verify they pass**

```bash
bash tests/test-lib.sh
```

Expected:
```
PASS: get_safe_scripts_dir: env override
PASS: get_safe_scripts_dir: default path
PASS: get_safe_scripts_dir: project config override
PASS: load_manifest: missing dir returns empty manifest
PASS: load_manifest: reads manifest file
PASS: is_heredoc python3 << 'EOF'
...
Results: 12 passed, 0 failed
```

- [ ] **Step 5: Commit**

```bash
git add scripts/lib.sh tests/test-lib.sh
git commit -m "feat: add shared lib.sh — config resolution, manifest load, pattern match"
```

---

### Task 3: SessionStart hook

**Files:**
- Create: `hooks/session-start`
- Create: `tests/test-session-start.sh`

- [ ] **Step 1: Write failing tests**

Write `tests/test-session-start.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
PASS=0; FAIL=0
SCRIPT_DIR="$(cd "$(dirname "$0")/.." && pwd)"

assert_eq() {
    if [ "$1" = "$2" ]; then PASS=$((PASS+1)); echo "PASS: $3"
    else FAIL=$((FAIL+1)); echo "FAIL: $3"; echo "  want: $2"; echo "  got:  $1"; fi
}
assert_contains() {
    if echo "$1" | grep -q "$2"; then PASS=$((PASS+1)); echo "PASS: contains '$2'"
    else FAIL=$((FAIL+1)); echo "FAIL: expected to contain '$2'"; echo "  in: $1"; fi
}

TMPDIR_TEST=$(mktemp -d)
trap "rm -rf '$TMPDIR_TEST'" EXIT

run_hook() {
    SAFE_SCRIPTS_DIR="$1" CLAUDE_PLUGIN_ROOT="$SCRIPT_DIR" \
        bash "${SCRIPT_DIR}/hooks/session-start"
}

# Test 1: empty safe-scripts dir → onboarding note
OUTPUT=$(run_hook "$TMPDIR_TEST/empty")
CONTEXT=$(echo "$OUTPUT" | jq -r '.hookSpecificOutput.additionalContext')
assert_contains "$CONTEXT" "safe-scripts"
assert_contains "$CONTEXT" "No safe scripts"

# Test 2: manifest with scripts → catalog injected
mkdir -p "$TMPDIR_TEST/with-scripts"
cat > "$TMPDIR_TEST/with-scripts/manifest.json" <<'JSON'
{
  "version": 1,
  "scripts": [
    {
      "name": "git-file-log",
      "description": "Show git history for a specific file",
      "script": "git-file-log.sh",
      "usage": "git-file-log <file> [--limit N]",
      "patterns": ["^git log"]
    }
  ]
}
JSON
OUTPUT=$(run_hook "$TMPDIR_TEST/with-scripts")
CONTEXT=$(echo "$OUTPUT" | jq -r '.hookSpecificOutput.additionalContext')
assert_contains "$CONTEXT" "git-file-log"
assert_contains "$CONTEXT" "Show git history"
assert_contains "$CONTEXT" "SAFE_SCRIPT_AVAILABLE"
assert_contains "$CONTEXT" "OFFER_SAFE_SCRIPT"

# Test 3: output is valid JSON
echo "$OUTPUT" | jq . > /dev/null
assert_eq "$?" "0" "session-start output is valid JSON"

echo ""
echo "Results: ${PASS} passed, ${FAIL} failed"
[ "$FAIL" -eq 0 ]
```

- [ ] **Step 2: Run tests — verify they fail**

```bash
bash tests/test-session-start.sh
```

Expected: `hooks/session-start: No such file or directory`

- [ ] **Step 3: Write hooks/session-start**

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
source "${SCRIPT_DIR}/../scripts/lib.sh"

SCRIPTS_DIR="$(get_safe_scripts_dir)"
MANIFEST="$(load_manifest "$SCRIPTS_DIR")"

SCRIPT_COUNT=$(echo "$MANIFEST" | jq '.scripts | length' 2>/dev/null || echo 0)

if [ "$SCRIPT_COUNT" -eq 0 ]; then
    CONTEXT="<safe-scripts>
No safe scripts saved yet. When a Bash command needs your approval, you will see an [OFFER_SAFE_SCRIPT] hint — follow the safe-scripts:safe-scripts skill to offer the user a generalized pre-approved version.
</safe-scripts>"
else
    CATALOG=$(echo "$MANIFEST" | jq -r \
        '.scripts[] | "- " + .usage + " — " + .description' 2>/dev/null)
    CONTEXT="<safe-scripts-catalog>
Available safe scripts — prefer these over raw bash (pre-approved, no permission dialog needed):
${CATALOG}

Rules:
- Before writing any Bash command, check this catalog. If a matching script exists, call it directly.
- When you see [SAFE_SCRIPT_AVAILABLE] in a hook response: do not retry the original command — use the safe script shown.
- When you see [HEREDOC_POSSIBLE_MATCH]: invoke safe-scripts:safe-scripts skill and compare the heredoc body to the candidate scripts before retrying.
- When you see [OFFER_SAFE_SCRIPT]: invoke safe-scripts:safe-scripts skill and offer the user the save choice in your next reply.
</safe-scripts-catalog>"
fi

emit_context "$CONTEXT" "SessionStart"
```

- [ ] **Step 4: Make executable**

```bash
chmod +x hooks/session-start
```

- [ ] **Step 5: Run tests — verify they pass**

```bash
bash tests/test-session-start.sh
```

Expected:
```
PASS: contains 'safe-scripts'
PASS: contains 'No safe scripts'
PASS: contains 'git-file-log'
PASS: contains 'Show git history'
PASS: contains 'SAFE_SCRIPT_AVAILABLE'
PASS: contains 'OFFER_SAFE_SCRIPT'
PASS: session-start output is valid JSON
Results: 7 passed, 0 failed
```

- [ ] **Step 6: Commit**

```bash
git add hooks/session-start tests/test-session-start.sh
git commit -m "feat: add session-start hook — inject safe-script catalog and behavioral rules"
```

---

### Task 4: PreToolUse hook — standard matching and heredoc two-tier

**Files:**
- Create: `hooks/pre-tool-use`
- Create: `tests/test-pre-tool-use.sh`

- [ ] **Step 1: Write failing tests**

Write `tests/test-pre-tool-use.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
PASS=0; FAIL=0
SCRIPT_DIR="$(cd "$(dirname "$0")/.." && pwd)"

assert_eq() {
    if [ "$1" = "$2" ]; then PASS=$((PASS+1)); echo "PASS: $3"
    else FAIL=$((FAIL+1)); echo "FAIL: $3"; echo "  want: $2"; echo "  got:  $1"; fi
}
assert_contains() {
    if echo "$1" | grep -q "$2"; then PASS=$((PASS+1)); echo "PASS: contains '$2'"
    else FAIL=$((FAIL+1)); echo "FAIL: expected '$2'"; echo "  in: $1"; fi
}
assert_empty() {
    if [ -z "$1" ]; then PASS=$((PASS+1)); echo "PASS: $2"
    else FAIL=$((FAIL+1)); echo "FAIL: $2 (expected empty, got '$1')"; fi
}

TMPDIR_TEST=$(mktemp -d)
trap "rm -rf '$TMPDIR_TEST'" EXIT

mkdir -p "$TMPDIR_TEST"
cat > "$TMPDIR_TEST/manifest.json" <<'JSON'
{
  "version": 1,
  "scripts": [
    {
      "name": "git-file-log",
      "description": "Show git history for a specific file",
      "script": "git-file-log.sh",
      "usage": "git-file-log <file> [--limit N]",
      "patterns": ["^git log (--oneline )?(-[0-9]+ )?-- .+"]
    },
    {
      "name": "analyze-csv",
      "description": "Analyze a CSV file with pandas",
      "script": "analyze-csv.py",
      "usage": "analyze-csv <file>",
      "patterns": ["^python3?\\s+<<\\s*'?EOF"],
      "heredoc": true
    }
  ]
}
JSON

run_hook() {
    local input="$1"
    echo "$input" | SAFE_SCRIPTS_DIR="$TMPDIR_TEST" CLAUDE_PLUGIN_ROOT="$SCRIPT_DIR" \
        bash "${SCRIPT_DIR}/hooks/pre-tool-use"
}

# Test 1: matching command → block decision with SAFE_SCRIPT_AVAILABLE
INPUT='{"tool_name":"Bash","tool_input":{"command":"git log --oneline -10 -- src/Button.tsx"}}'
OUTPUT=$(run_hook "$INPUT")
assert_eq "$(echo "$OUTPUT" | jq -r '.decision')" "block" "standard match: decision is block"
assert_contains "$(echo "$OUTPUT" | jq -r '.reason')" "SAFE_SCRIPT_AVAILABLE" "standard match: reason contains marker"
assert_contains "$(echo "$OUTPUT" | jq -r '.reason')" "git-file-log" "standard match: reason names the script"

# Test 2: non-matching command → empty output (pass-through)
INPUT='{"tool_name":"Bash","tool_input":{"command":"ls -la"}}'
OUTPUT=$(run_hook "$INPUT")
assert_empty "$OUTPUT" "no match: empty output (pass-through)"

# Test 3: heredoc with candidates → block with HEREDOC_POSSIBLE_MATCH
INPUT=$(printf '{"tool_name":"Bash","tool_input":{"command":"python3 << '"'"'EOF'"'"'\nimport pandas as pd\nEOF"}}')
OUTPUT=$(run_hook "$INPUT")
assert_eq "$(echo "$OUTPUT" | jq -r '.decision')" "block" "heredoc: decision is block"
assert_contains "$(echo "$OUTPUT" | jq -r '.reason')" "HEREDOC_POSSIBLE_MATCH" "heredoc: reason contains marker"
assert_contains "$(echo "$OUTPUT" | jq -r '.reason')" "analyze-csv" "heredoc: reason lists candidate"

# Test 4: heredoc with no candidates → pass-through
EMPTY_MANIFEST='{"version":1,"scripts":[]}'
INPUT=$(printf '{"tool_name":"Bash","tool_input":{"command":"python3 << '"'"'EOF'"'"'\nprint(1)\nEOF"}}')
OUTPUT=$(echo "$INPUT" | SAFE_SCRIPTS_DIR="$TMPDIR_TEST/empty" CLAUDE_PLUGIN_ROOT="$SCRIPT_DIR" \
    bash "${SCRIPT_DIR}/hooks/pre-tool-use")
assert_empty "$OUTPUT" "heredoc with no candidates: pass-through"

# Test 5: missing command field → pass-through
INPUT='{"tool_name":"Bash","tool_input":{}}'
OUTPUT=$(run_hook "$INPUT")
assert_empty "$OUTPUT" "missing command: pass-through"

# Test 6: output is valid JSON when non-empty
INPUT='{"tool_name":"Bash","tool_input":{"command":"git log --oneline -10 -- src/Button.tsx"}}'
OUTPUT=$(run_hook "$INPUT")
echo "$OUTPUT" | jq . > /dev/null
assert_eq "$?" "0" "block output is valid JSON"

echo ""
echo "Results: ${PASS} passed, ${FAIL} failed"
[ "$FAIL" -eq 0 ]
```

- [ ] **Step 2: Run tests — verify they fail**

```bash
bash tests/test-pre-tool-use.sh
```

Expected: `hooks/pre-tool-use: No such file or directory`

- [ ] **Step 3: Write hooks/pre-tool-use**

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
source "${SCRIPT_DIR}/../scripts/lib.sh"

INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // empty' 2>/dev/null)

[ -z "$COMMAND" ] && exit 0

SCRIPTS_DIR="$(get_safe_scripts_dir)"
MANIFEST="$(load_manifest "$SCRIPTS_DIR")"

# Heredoc path: structural match → two-tier (semantic done by skill)
if is_heredoc "$COMMAND"; then
    CANDIDATES=$(find_heredoc_candidates "$MANIFEST")
    if [ -n "$CANDIDATES" ]; then
        REASON="[HEREDOC_POSSIBLE_MATCH] This command contains an inline script. The following saved safe scripts may cover it:
${CANDIDATES}
Invoke the safe-scripts:safe-scripts skill. Compare the heredoc body to each candidate's script file in ${SCRIPTS_DIR}. If a match is found, use that safe script with mapped arguments. If no match, offer to save a new safe script."
        jq -n --arg r "$REASON" '{"decision":"block","reason":$r}'
    fi
    exit 0
fi

# Standard path: regex pattern match
MATCH=$(find_matching_script "$COMMAND" "$MANIFEST")
[ -z "$MATCH" ] && exit 0

NAME=$(echo "$MATCH" | jq -r '.name')
USAGE=$(echo "$MATCH" | jq -r '.usage')
SCRIPT=$(echo "$MATCH" | jq -r '.script')
DESC=$(echo "$MATCH" | jq -r '.description')
FULL_PATH="${SCRIPTS_DIR}/${SCRIPT}"

REASON="[SAFE_SCRIPT_AVAILABLE] name=${NAME} description='${DESC}' usage='${USAGE}' path='${FULL_PATH}'
Do not retry the original command. Call the safe script at the path above, mapping arguments from the original command to the usage signature."

jq -n --arg r "$REASON" '{"decision":"block","reason":$r}'
```

- [ ] **Step 4: Make executable**

```bash
chmod +x hooks/pre-tool-use
```

- [ ] **Step 5: Run tests — verify they pass**

```bash
bash tests/test-pre-tool-use.sh
```

Expected:
```
PASS: standard match: decision is block
PASS: standard match: reason contains marker
PASS: standard match: reason names the script
PASS: no match: empty output (pass-through)
PASS: heredoc: decision is block
PASS: heredoc: reason contains marker
PASS: heredoc: reason lists candidate
PASS: heredoc with no candidates: pass-through
PASS: missing command: pass-through
PASS: block output is valid JSON
Results: 10 passed, 0 failed
```

- [ ] **Step 6: Commit**

```bash
git add hooks/pre-tool-use tests/test-pre-tool-use.sh
git commit -m "feat: add pre-tool-use hook — manifest match with redirect, heredoc two-tier matching"
```

---

### Task 5: PermissionRequest hook

**Files:**
- Create: `hooks/permission-request`
- Create: `tests/test-permission-request.sh`

- [ ] **Step 1: Write failing tests**

Write `tests/test-permission-request.sh`:

```bash
#!/usr/bin/env bash
set -euo pipefail
PASS=0; FAIL=0
SCRIPT_DIR="$(cd "$(dirname "$0")/.." && pwd)"

assert_eq() {
    if [ "$1" = "$2" ]; then PASS=$((PASS+1)); echo "PASS: $3"
    else FAIL=$((FAIL+1)); echo "FAIL: $3"; echo "  want: $2"; echo "  got:  $1"; fi
}
assert_contains() {
    if echo "$1" | grep -q "$2"; then PASS=$((PASS+1)); echo "PASS: contains '$2'"
    else FAIL=$((FAIL+1)); echo "FAIL: expected '$2'"; echo "  in: $1"; fi
}

run_hook() {
    local input="$1"
    echo "$input" | CLAUDE_PLUGIN_ROOT="$SCRIPT_DIR" \
        bash "${SCRIPT_DIR}/hooks/permission-request"
}

# Test 1: standard command → OFFER_SAFE_SCRIPT injected
INPUT='{"tool_name":"Bash","tool_input":{"command":"git log --oneline -10 -- src/Button.tsx"}}'
OUTPUT=$(run_hook "$INPUT")
CONTEXT=$(echo "$OUTPUT" | jq -r '.hookSpecificOutput.additionalContext')
assert_contains "$CONTEXT" "OFFER_SAFE_SCRIPT" "injects OFFER_SAFE_SCRIPT marker"
assert_contains "$CONTEXT" "safe-scripts:safe-scripts" "instructs Claude to invoke skill"

# Test 2: output is valid JSON
echo "$OUTPUT" | jq . > /dev/null
assert_eq "$?" "0" "output is valid JSON"

# Test 3: missing command → still emits context (command may be in description)
INPUT='{"tool_name":"Bash","tool_input":{}}'
OUTPUT=$(run_hook "$INPUT")
assert_contains "$(echo "$OUTPUT" | jq -r '.hookSpecificOutput.additionalContext')" \
    "OFFER_SAFE_SCRIPT" "missing command still emits offer"

echo ""
echo "Results: ${PASS} passed, ${FAIL} failed"
[ "$FAIL" -eq 0 ]
```

- [ ] **Step 2: Run tests — verify they fail**

```bash
bash tests/test-permission-request.sh
```

Expected: `hooks/permission-request: No such file or directory`

- [ ] **Step 3: Write hooks/permission-request**

```bash
#!/usr/bin/env bash
set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
source "${SCRIPT_DIR}/../scripts/lib.sh"

INPUT=$(cat)
COMMAND=$(echo "$INPUT" | jq -r '.tool_input.command // empty' 2>/dev/null)

# Truncate long commands for readability in the injected context
DISPLAY_CMD="${COMMAND:0:200}"
[ "${#COMMAND}" -gt 200 ] && DISPLAY_CMD="${DISPLAY_CMD}..."

CONTEXT="[OFFER_SAFE_SCRIPT] A Bash command needs your approval: ${DISPLAY_CMD}

Invoke the safe-scripts:safe-scripts skill. In your next reply, offer the user:
(a) Save a generalized safe script — permanent auto-approval for this and similar commands
(b) Approve once — run as-is this time only"

emit_context "$CONTEXT" "PermissionRequest"
```

- [ ] **Step 4: Make executable**

```bash
chmod +x hooks/permission-request
```

- [ ] **Step 5: Run tests — verify they pass**

```bash
bash tests/test-permission-request.sh
```

Expected:
```
PASS: contains 'OFFER_SAFE_SCRIPT'
PASS: contains 'safe-scripts:safe-scripts'
PASS: output is valid JSON
PASS: contains 'OFFER_SAFE_SCRIPT'
Results: 4 passed, 0 failed
```

- [ ] **Step 6: Commit**

```bash
git add hooks/permission-request tests/test-permission-request.sh
git commit -m "feat: add permission-request hook — inject OFFER_SAFE_SCRIPT hint"
```

---

### Task 6: Behavior-shaping skill — skills/safe-scripts/SKILL.md

**Files:**
- Create: `skills/safe-scripts/SKILL.md`

- [ ] **Step 1: Write skills/safe-scripts/SKILL.md**

```markdown
# safe-scripts

Use this skill when:
- You see `[SAFE_SCRIPT_AVAILABLE]`, `[HEREDOC_POSSIBLE_MATCH]`, or `[OFFER_SAFE_SCRIPT]` in a hook response
- You are about to write a Bash command and the `<safe-scripts-catalog>` block is in your context
- The user asks to save a command as a safe script

---

## Proactive Preference

Before writing any `Bash` tool call, scan the `<safe-scripts-catalog>` block in your context. If a listed script covers the task, call it directly with a fully-qualified path. Do not write a raw bash equivalent.

**Argument mapping:** translate hardcoded values from your intended command to the script's usage signature.

Example:
- Intended: `git log --oneline -5 -- src/App.tsx`
- Catalog: `git-file-log <file> [--limit N] — Show git history for a specific file`
- Call: `/home/user/.claude/safe-scripts/git-file-log.sh src/App.tsx --limit 5`

---

## On [SAFE_SCRIPT_AVAILABLE]

The hook blocked a command because a safe script matches. Do NOT retry the original command.

1. Parse the injection: extract `name`, `usage`, `path`, `description`
2. Map the arguments from the blocked command to the usage signature
3. Call the safe script at `path` with the mapped arguments
4. Do not explain the redirect to the user — just proceed

---

## On [HEREDOC_POSSIBLE_MATCH]

The hook blocked an inline heredoc script and listed candidate safe scripts.

1. For each candidate listed: read its script file from the safe-scripts directory
2. Compare the heredoc body to the candidate's content (same logic? same structure? parameterizable overlap?)
3. If strong match: call the safe script with mapped arguments (do not retry the heredoc)
4. If no match: follow the Save Procedure below, using the heredoc body as the script source

---

## On [OFFER_SAFE_SCRIPT]

A command was just approved or ran. In your **next reply**, before anything else, present this choice:

> "That command ran successfully. I can save a generalized version as a safe script — future similar commands would be auto-approved with no dialog. Want me to:
> **(a) Save as a safe script** — I'll generalize it, show you the script, and save it permanently
> **(b) Leave it** — continue running this command with approval each time"

Wait for the user's response. If (a): follow the Save Procedure. If (b): continue.

---

## Save Procedure

### 1. Detect interpreter and form

| Command form | Interpreter | Shebang | Extension |
|---|---|---|---|
| `git ...`, `ls ...`, raw shell | bash | `#!/bin/bash` | `.sh` |
| `python3 ...` or heredoc with `import` | Python | `#!/usr/bin/env python3` | `.py` |
| `node ...` or heredoc with `require`/`const` | Node | `#!/usr/bin/env node` | `.js` |
| `perl ...` | Perl | `#!/usr/bin/env perl` | `.pl` |
| `go run ...`, `npx ...`, `npx ts-node ...` | bash wrapper | `#!/bin/bash` | `.sh` |
| Heredoc: extract the body, detect language from content | (as above) | (as above) | (as above) |

### 2. Generalize

Replace every hardcoded value with a parameter:
- File paths → positional args (`$1`, `$FILE`, `sys.argv[1]`, `process.argv[2]`)
- Numeric limits → named flags with defaults (`--limit N`, default 20)
- Branch names, filter strings, env names → named args

Add a usage comment at the top. Use argument parsing idiomatic to the language:
- Bash: `${1:?Usage: script <arg>}` for required, `${2:-default}` for optional
- Python: `argparse` or `sys.argv`
- Node: `process.argv`
- Perl: positional `$ARGV[0]`

For heredoc sources: set `"heredoc": true` in the manifest entry and use a structural pattern (e.g., `^python3?\s+<<\s*'?EOF`).

### 3. Choose a name

Verb-noun kebab-case: `git-file-log`, `analyze-csv`, `find-in-files`, `run-tests`. Extension matches interpreter.

### 4. Show the user the script and wait for confirmation

Present the complete script and its name. Wait for the user to confirm. Only proceed to Step 5 after explicit confirmation ("yes", "save it", "looks good", etc.).

**Example presentation:**
> ```
> Name: git-file-log.sh
> Description: Show git history for a specific file
>
> #!/bin/bash
> # Usage: git-file-log <file> [--limit N]
> set -euo pipefail
> FILE="${1:?Usage: git-file-log <file> [--limit N]}"
> LIMIT="${2:---limit}"
> LIMIT_VAL="${3:-20}"
> if [ "$LIMIT" = "--limit" ]; then
>     git log --oneline "-${LIMIT_VAL}" -- "$FILE"
> else
>     git log --oneline -- "$FILE"
> fi
> ```

### 5. Write the files (after confirmation)

Run these Bash commands in order:

**a. Resolve the safe-scripts directory:**
```bash
SCRIPTS_DIR="${HOME}/.claude/safe-scripts"
# Or read from .claude/safe-scripts-config.json if present:
# SCRIPTS_DIR=$(jq -r '.safe_scripts_dir // empty' .claude/safe-scripts-config.json 2>/dev/null || echo "$SCRIPTS_DIR")
mkdir -p "$SCRIPTS_DIR"
```

**b. Write the script file:**
```bash
cat > "${SCRIPTS_DIR}/<name>" << 'EOF'
<script content>
EOF
chmod +x "${SCRIPTS_DIR}/<name>"
```

**c. Update manifest.json** — read existing, append entry, write back:
```bash
MANIFEST="${SCRIPTS_DIR}/manifest.json"
EXISTING=$([ -f "$MANIFEST" ] && cat "$MANIFEST" || echo '{"version":1,"scripts":[]}')
echo "$EXISTING" | jq --arg name "<name>" \
    --arg desc "<description>" \
    --arg script "<filename>" \
    --arg usage "<usage>" \
    --argjson patterns '["<pattern1>","<pattern2>"]' \
    --arg added "$(date +%Y-%m-%d)" \
    '.scripts += [{"name":$name,"description":$desc,"script":$script,"usage":$usage,"patterns":$patterns,"added":$added}]' \
    > "${MANIFEST}.tmp" && mv "${MANIFEST}.tmp" "$MANIFEST"
```

For heredoc-originated scripts, also pass `--argjson heredoc 'true'` and include `"heredoc":$heredoc` in the object.

**d. Update settings.json allow-list** — add the script path to the nearest settings.json:
```bash
# Use project-local settings if it exists, else user settings
SETTINGS=".claude/settings.json"
[ ! -f "$SETTINGS" ] && SETTINGS="${HOME}/.claude/settings.json"
ALLOW_ENTRY="Bash(${SCRIPTS_DIR}/<name>*)"
EXISTING_SETTINGS=$([ -f "$SETTINGS" ] && cat "$SETTINGS" || echo '{}')
echo "$EXISTING_SETTINGS" | jq --arg entry "$ALLOW_ENTRY" \
    '.permissions.allow = ((.permissions.allow // []) + [$entry] | unique)' \
    > "${SETTINGS}.tmp" && mv "${SETTINGS}.tmp" "$SETTINGS"
```

### 6. Self-note and re-attempt

After all files are written, say in your reply:
> "Saved as `<name>`. I'll use it for the rest of this session whenever I'd otherwise write an equivalent command."

Then immediately re-attempt the original task using the new safe script.

---

## Within-Session Discovery

Scripts saved mid-session are covered three ways:
1. **Self-note** (above) — Claude remembers proactively
2. **PreToolUse hook** — reads manifest.json fresh from disk on every call; a script saved moments ago is interceptable on the next command
3. **SessionStart** — all future sessions see the full updated catalog
```

- [ ] **Step 2: Verify the skill file contains all required markers**

```bash
for marker in SAFE_SCRIPT_AVAILABLE HEREDOC_POSSIBLE_MATCH OFFER_SAFE_SCRIPT \
              "Save Procedure" "Proactive Preference" "Within-Session"; do
    grep -q "$marker" skills/safe-scripts/SKILL.md \
        && echo "PASS: contains '$marker'" \
        || echo "FAIL: missing '$marker'"
done
```

Expected: 6 × `PASS`

- [ ] **Step 3: Commit**

```bash
git add skills/safe-scripts/SKILL.md
git commit -m "feat: add safe-scripts skill — proactive preference, redirect, save procedure"
```

---

### Task 7: Test runner, README, LICENSE, and smoke test

**Files:**
- Create: `tests/run-tests.sh`
- Create: `README.md`
- Create: `LICENSE`

- [ ] **Step 1: Write tests/run-tests.sh**

```bash
#!/usr/bin/env bash
set -euo pipefail
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"

TOTAL_PASS=0
TOTAL_FAIL=0

run_suite() {
    local name="$1"
    local file="$2"
    echo "=== $name ==="
    if bash "$file"; then
        TOTAL_PASS=$((TOTAL_PASS+1))
    else
        TOTAL_FAIL=$((TOTAL_FAIL+1))
    fi
    echo ""
}

run_suite "lib.sh"           "${SCRIPT_DIR}/test-lib.sh"
run_suite "session-start"    "${SCRIPT_DIR}/test-session-start.sh"
run_suite "pre-tool-use"     "${SCRIPT_DIR}/test-pre-tool-use.sh"
run_suite "permission-request" "${SCRIPT_DIR}/test-permission-request.sh"

echo "=== Suite Results: ${TOTAL_PASS} passed, ${TOTAL_FAIL} failed ==="
[ "$TOTAL_FAIL" -eq 0 ]
```

- [ ] **Step 2: Run the full test suite**

```bash
bash tests/run-tests.sh
```

Expected:
```
=== lib.sh ===
...
Results: 12 passed, 0 failed

=== session-start ===
...
Results: 7 passed, 0 failed

=== pre-tool-use ===
...
Results: 10 passed, 0 failed

=== permission-request ===
...
Results: 4 passed, 0 failed

=== Suite Results: 4 passed, 0 failed ===
```

- [ ] **Step 3: Write README.md**

```markdown
# safe-scripts

A Claude Code plugin that builds a personal library of pre-approved safe scripts to eliminate repetitive permission dialogs.

## How it works

When Claude runs a Bash command that needs approval, the plugin intercepts and offers to save a generalized, parameterized version as a "safe script." Once saved, that script is permanently pre-approved — no more dialogs for that class of command.

## Requirements

- Claude Code with plugin support
- `jq` ≥ 1.5 (`brew install jq` / `apt install jq`)
- bash ≥ 3.2

## Installation

```bash
claude plugin install <repo-url>
```

## Configuration

By default, scripts are saved to `~/.claude/safe-scripts/`.

To override per-project, create `.claude/safe-scripts-config.json`:

```json
{
  "safe_scripts_dir": "./.claude/safe-scripts"
}
```

A project-local directory can be committed to git for team-shared script libraries.

## Supported interpreters

bash, python3, node, perl, go (via bash wrapper), npx/ts-node (via bash wrapper), and inline heredoc scripts for any of the above.

## License

MIT
```

- [ ] **Step 4: Write LICENSE**

```
MIT License

Copyright (c) 2026

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```

- [ ] **Step 5: Final full test suite run**

```bash
bash tests/run-tests.sh
```

Expected: `Suite Results: 4 passed, 0 failed`

- [ ] **Step 6: Commit**

```bash
git add tests/run-tests.sh README.md LICENSE
git commit -m "feat: add test runner, README, LICENSE — plugin complete"
```
