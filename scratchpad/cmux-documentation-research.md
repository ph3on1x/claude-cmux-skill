# cmux Documentation & Research

> Last updated by /write on 2026-03-25
> Source: Research session (3 parallel agents) + skill evaluation & fixes (v1.3.0) + hooks simplification (v1.5.0) + panes-as-default fix (v1.5.1) + balanced splits & CLI fix (v1.6.0) + agent cleanup & output persistence (v1.7.0) + prompt file fix (v1.8.0 → v1.8.1) + interactive mode fix (v1.9.0)

## Summary

Comprehensive research on cmux — a native macOS terminal application (Swift/AppKit + libghostty) built for AI coding agents. Covers installation, full CLI reference (220+ commands), Claude Code integration via plugins/hooks/skills, browser automation, multi-agent orchestration patterns, and troubleshooting. Research produced 3 detailed files totaling ~1,700 lines.

## Overview

- **What**: Free, open-source native macOS terminal for AI coding agents (AGPL-3.0)
- **Tech stack**: Swift/AppKit, libghostty (GPU-accelerated rendering), WKWebView (browser), Bonsplit (layout), Sparkle (auto-update)
- **Requirements**: macOS 14.0 (Sonoma)+, Apple Silicon or Intel
- **Version**: 0.62.2+ (as of research date)
- **GitHub**: manaflow-ai/cmux (~9.8k stars, launched Feb 2026)
- **Works with**: Claude Code, Codex, OpenCode, Gemini CLI, Kiro, Aider, any CLI tool

## Installation

### Homebrew (recommended)
```bash
brew tap manaflow-ai/cmux
brew install --cask cmux
brew upgrade --cask cmux  # update
```

### DMG
Download from GitHub releases or cmux.app, drag to Applications. Auto-updates via Sparkle.

### CLI symlink (for use outside cmux terminals)
```bash
sudo ln -sf "/Applications/cmux.app/Contents/Resources/bin/cmux" /usr/local/bin/cmux
```

## Architecture

**Hierarchy**: Window -> Workspace (sidebar tab) -> Pane (split region) -> Surface (terminal or browser tab)

**Short refs format**: `workspace:2`, `pane:1`, `surface:7` — always prefer over UUIDs.

**Communication**: CLI sends newline-terminated JSON to a Unix domain socket. Release socket at `/tmp/cmux.sock`, debug at `/tmp/cmux-debug.sock`.

**Socket access modes**:
| Mode | Description |
|------|-------------|
| `off` | Socket disabled |
| `cmuxOnly` | Only cmux-spawned processes (default) |
| `allowAll` | Any local process |

## Environment Variables

### Auto-set inside cmux terminals
| Variable | Purpose |
|----------|---------|
| `CMUX_SOCKET_PATH` | Control socket (CLI auto-detects) |
| `CMUX_WORKSPACE_ID` | Current workspace UUID |
| `CMUX_SURFACE_ID` | Current surface UUID |
| `TERM_PROGRAM` | Set to `"ghostty"` |
| `TERM` | Set to `"xterm-ghostty"` |

### Override variables
| Variable | Purpose |
|----------|---------|
| `CMUX_SOCKET_ENABLE` | Force enable/disable (`1`/`0`) |
| `CMUX_SOCKET_MODE` | Override access mode |
| `CMUX_SOCKET_PASSWORD` | Socket auth password |

## Configuration

Reads from Ghostty config files: `~/.config/ghostty/config` or `~/.ghostty.config`. Configures fonts, colors/themes, keybindings, scrollback, working directory defaults, split pane styling.

cmux-specific shortcuts configurable in Settings -> Keyboard Shortcuts.

```bash
cmux themes list          # List available themes
cmux themes set <name>    # Apply a theme
cmux themes clear         # Reset to default
```

## CLI Command Reference (Key Commands)

### System / Meta
- `cmux version`, `cmux ping`, `cmux capabilities`, `cmux identify --json`, `cmux tree --json`

### Window/Workspace/Pane/Surface Management
- `cmux new-workspace [--cwd P] [--command T]` — create workspace
- `cmux select-workspace --workspace W` — switch
- `cmux rename-workspace "name"` — rename
- `cmux new-split right|down` — split pane
- `cmux new-pane [--type terminal|browser] [--direction ...]` — create pane
- `cmux close-surface --surface S` — close surface
- `cmux resize-pane`, `cmux swap-pane`, `cmux break-pane`, `cmux join-pane`
- `cmux find-window --content "error" --select` — search terminal content

### Terminal I/O
- `cmux send --surface S "text\n"` — send text (\\n = Enter)
- `cmux send-key --surface S ctrl+c` — send key event
- `cmux read-screen --surface S [--scrollback] [--lines N]` — read output
- `cmux pipe-pane --surface S --command "grep ..."` — pipe through command

### Sidebar Metadata
- `cmux set-status "key" "value" [--icon I] [--color C]` — status pill
- `cmux set-progress 0.5 [--label "text"]` — progress bar
- `cmux log "message" [--level info|success|warning|error]` — log entry
- `cmux sidebar-state` — dump all sidebar metadata
- `cmux clear-status`, `cmux clear-progress`, `cmux clear-log`

### Synchronization & Buffers
- `cmux wait-for --signal <name>` — signal completion
- `cmux wait-for <name> --timeout S` — wait for signal
- `cmux set-buffer --name "key" "value"` — store data
- `cmux paste-buffer --name "key" --surface S` — paste into surface

### Notifications
- `cmux notify --title T [--body B]` — in-app notification
- Supports OSC 9, OSC 99 (Kitty), OSC 777 (RXVT) protocols
- Custom notification command configurable in Settings with env vars `CMUX_NOTIFICATION_TITLE/SUBTITLE/BODY`

### Browser Automation
- `cmux browser open <url> --json` — open browser surface
- `cmux browser surface:N snapshot --interactive` — get element refs (ALWAYS use `--interactive`)
- `cmux browser surface:N click e2` — interact via refs (NEVER CSS selectors)
- `cmux browser surface:N fill e3 "text"` — fill input
- `cmux browser surface:N wait --load-state complete` — wait for navigation
- `cmux browser surface:N eval 'JS expression'` — execute JavaScript
- `cmux browser surface:N state save/load <path>` — persist browser state
- `cmux browser surface:N console list` / `errors list` — diagnostics

### Claude Code Integration
- `cmux claude-hook session-start` — mark active
- `cmux claude-hook stop` — mark idle
- `cmux claude-hook notification` — forward notification
- `cmux claude-hook prompt-submit` — handle prompt

## Claude Code Integration Details

### Detection mechanism
`CMUX_SOCKET_PATH` env var — plugins/skills check this to determine if running inside cmux.

### The Agent Tool Blocking Hook
Located in `hooks/hooks.json` of the claude-cmux-skill plugin:
- **Event**: PreToolUse, **Matcher**: `Agent`
- **Behavior**: When `CMUX_SOCKET_PATH` is set, denies the Agent tool with a reason explaining to use cmux panes instead
- **Why**: Built-in Agent tool spawns invisible background processes. cmux panes give each agent a real isolated terminal with visibility, monitoring (`read-screen`), sidebar tracking, and inter-pane coordination

### The Redirect Pattern
1. User asks to "run 3 agents in parallel"
2. Claude attempts Agent tool -> PreToolUse hook denies
3. Denial reason redirects to `using-cmux` skill
4. Skill provides cmux CLI commands for split panes + launch

### Plugin Structure (claude-cmux-skill v1.9.0)
```
claude-cmux-skill/
├── .claude-plugin/plugin.json
├── hooks/hooks.json           # PreToolUse Agent blocking hook
├── skills/using-cmux/
│   ├── SKILL.md               # Core skill (auto-triggers)
│   └── references/
│       ├── orchestration.md
│       ├── browser-automation.md
│       ├── notifications.md
│       └── complete-cli.md
```

### Claude Code Hook Events (22 total)
Key events: SessionStart, UserPromptSubmit, PreToolUse, PostToolUse, Notification, SubagentStart/Stop, Stop, PreCompact, SessionEnd. PreToolUse can return `allow|deny|ask` decisions.

## Multi-Agent Orchestration Best Practices

### Core Pattern
Decompose -> Isolate -> Launch -> Monitor -> Collect -> Clean Up

### Practical Limits
- **5-7 concurrent agents** is the ceiling on a laptop (API rate limits, merge conflicts, review bottleneck)
- **Breakeven**: ~30+ min of sequential work. Tasks <15 min single-agent -> multi-agent is slower (5-10 min setup overhead)
- 3 Claude Code instances on Pro plan exhaust rate limits ~3x faster

### Task Decomposition Rules
- Only parallelize tasks with no dependencies on each other's outputs
- Good: separate files/modules per agent
- Bad: Agent B imports something Agent A hasn't created yet
- Use git worktrees for file isolation when agents share a codebase

### Git Worktrees for Isolation
```bash
git worktree add ../settings-api feature/settings-api
git worktree add ../settings-ui feature/settings-ui
```

### Error Recovery
```bash
cmux send-key --surface surface:N ctrl+c    # Interrupt
cmux clear-history --surface surface:N      # Clear
cmux send --surface surface:N "claude -p 'retry task'\n"  # Restart
```

### Comparison: Agent Tool vs cmux Panes
| Aspect | Built-in Agent Tool | cmux Panes |
|--------|-------------------|------------|
| Isolation | Shared process | Separate processes |
| Visibility | Hidden (background) | Visible in split panes |
| Monitoring | Wait for completion | `read-screen` anytime |
| Communication | Return value only | Buffers, sync tokens, terminal I/O |
| Progress tracking | None | Sidebar status/progress/logs |
| Browser support | No | Yes |

## Keyboard Shortcuts (Key)

| Action | Shortcut |
|--------|----------|
| New workspace | Cmd+N |
| Close workspace | Cmd+Shift+W |
| Split right | Cmd+D |
| Split down | Cmd+Shift+D |
| Focus pane directionally | Opt+Cmd+arrows |
| Open browser | Cmd+Shift+L |
| Notifications panel | Cmd+Shift+I |
| Jump to unread | Cmd+Shift+U |

## Browser Automation Key Rules

1. **Always** use `snapshot --interactive` to get element refs
2. **Never** use CSS selectors directly — use refs (`e1`, `e2`, `e3`)
3. **Always** re-snapshot after navigation (DOM changes invalidate refs)
4. **Always** `wait --load-state complete` after page loads
5. Use high-level commands (`click`, `fill`, `type`) not low-level `input_*`

### WKWebView Limitations (not supported)
Viewport emulation, geolocation faking, network interception, video recording, tracing, low-level input commands

## Notification Strategy

| Need | Use |
|------|-----|
| User is in cmux | `cmux notify` |
| User may be in another app | `osascript -e 'display notification ...'` with sound |
| Context-aware (pane/workspace) | `cmux notify` |
| Must persist after cmux closes | `osascript` |

Notifications suppressed when: cmux window focused, workspace active, notification panel open.

## Troubleshooting

- **"Failed to connect to socket"**: Sandbox blocking `/tmp/cmux.sock`. Check `cmux ping`.
- **Diagnostic commands**: `cmux ping`, `cmux identify --json`, `cmux tree --json`, `cmux surface-health`, `cmux capabilities`, `cmux sidebar-state`
- **Stuck agent**: `read-screen --lines 10` to check state, `pipe-pane --command "grep -c error"` to filter, `send-key ctrl+c` to interrupt
- **Session restore limitation**: Does NOT restore active processes (Claude Code, vim, etc.) — only layout, directories, scrollback, browser history
- **Socket auth**: `cmux --password <pw>` or `CMUX_SOCKET_PASSWORD` env var

## Known Limitations

1. macOS only — no Linux/Windows
2. No session restore for active processes
3. WKWebView limitations (no viewport emulation, network interception, etc.)
4. Sandbox may block socket access
5. Very new (launched Feb 2026), rapidly evolving

## Related Projects

| Project | Description |
|---------|-------------|
| cmuxlayer | MCP server for cmux workspace orchestration |
| cmux-skill (hashangit) | Another cmux skill implementation |
| cmux-agent-mcp | MCP server for multi-agent cognition |
| CodeMux | Cross-platform terminal multiplexer for AI |
| amux | tmux-based parallel AI agent runner |

## Hooks Simplification (v1.5.0)

**Problem (v1.4.0)**: The v1.4.0 fix added custom `cmux set-status` calls (via `cmux identify --json | awk`) to the SessionStart and Stop hooks to set surface-ref-keyed status pills. This caused two issues:
1. **Duplicate status pills** — `cmux claude-hook session-start/stop` already manages sidebar status natively (e.g., "Needs input"). The custom `set-status` added a second pill, causing conflicting displays like "Needs input" + "done" on a single pane.
2. **Hook exit code failures** — When `cmux identify --json` failed to extract the surface ref (`$SREF` empty), `[ -n "$SREF" ]` returned exit code 1, causing hook errors visible to the LLM. This corrupted the agent's context and led to hallucinated commands like `cmux pane create` (which doesn't exist).

**Root cause**: The custom `set-status` logic was redundant — `cmux claude-hook` already handles all sidebar lifecycle status natively.

**Fix**: Removed all `cmux identify --json | awk` + `cmux set-status` logic from SessionStart and Stop hooks. Reverted to simple form:
```bash
if [ -n "$CMUX_SOCKET_PATH" ]; then cmux claude-hook session-start; else exit 0; fi
if [ -n "$CMUX_SOCKET_PATH" ]; then cmux claude-hook stop; else exit 0; fi
```

**Skill changes**: Reverted status key guidance from `"surface:N"` back to descriptive keys (`"agent-1"`, `"agent-2"`) in `SKILL.md` and `orchestration.md`. Added `cmux pane create` to Common Mistakes table.

**Result**: Single status pill per pane (managed by `cmux claude-hook`), no hook exit code errors, no hallucinated commands.

## Panes-as-Default Fix (v1.5.1)

**Problem**: Inconsistent agent-spawning behavior across models — Opus correctly used `cmux new-split` (panes within the current workspace), while Sonnet created new workspaces and struggled to orchestrate them. Root cause: the skill documented both patterns with equal weight and no explicit hierarchy of preference.

**Specific gaps**:
1. `SKILL.md` "Agent Orchestration" section had no "prefer panes" statement
2. `SKILL.md` "Common Mistakes" table had no entry warning against workspace-as-default
3. `orchestration.md` "Workspace-Per-Project Pattern" section wasn't clearly framed as a special case — weaker models treated it as the general pattern

**Fix**: 3 targeted edits:
1. `SKILL.md` — Added `**Default: panes, not workspaces.**` callout immediately after the Agent Orchestration section header
2. `SKILL.md` — Added row to Common Mistakes table warning against workspaces for parallel agents
3. `orchestration.md` — Added `> **Special case only.**` callout under "Workspace-Per-Project Pattern" heading

**Result**: Explicit panes-first default visible to all models at the first decision point. Workspace pattern clearly gated as a special case.

## Balanced Splits & CLI Fix (v1.6.0)

**Problem 1 — Uneven pane splitting**: Users reported some panes being very large and others very small. Root cause: Claude splits whichever pane has focus (the most recently created one), leading to recursive halving of one pane (50% → 25% → 12.5% → 6.25%) while the original pane stays untouched.

**Problem 2 — Wrong CLI invocation**: All skill examples used `claude 'prompt'` (interactive mode) instead of `claude -p 'prompt'` (non-interactive print mode). Without `-p`: the workspace trust dialog may block execution, the agent starts an interactive session waiting for input, and Claude often "improvises" by saving prompts to temp files (`/tmp/agent-1.md`, `/tmp/agent-1.sh`) and piping them.

**Fix — Balanced splits**:
1. `SKILL.md` — Replaced "Creating Panes" with "Creating Balanced Pane Layouts": concrete recipes for 1, 2, 3, and 4+ agents using surface ref capture (`awk '{print $2}'`) and `--surface` targeting
2. `orchestration.md` — Replaced "Plan and Create Panes" with balanced 2×2 grid recipe using same pattern
3. Key principle: always capture surface refs from `new-split` output, always use `--surface $REF` for subsequent splits
4. Added Common Mistakes entry: "Splitting the focused pane recursively"

**Fix — CLI invocation**:
1. Changed every `claude 'prompt'` → `claude -p 'prompt'` across SKILL.md, orchestration.md, README.md
2. Added bold instructions: "Always use `claude -p 'prompt'`" and "Never save prompts to temp files"
3. Added Common Mistakes entries for missing `-p` flag and temp file patterns
4. Added single-quote escaping example for complex prompts

**Layout recipes (from SKILL.md)**:
- **1 agent**: `S1=$(cmux new-split right | awk '{print $2}')` → [orch 50% | agent 50%]
- **2 agents**: split right + subdivide → [orch 50% | agent-1 25% / agent-2 25%]
- **3 agents**: 2×2 grid using `$ORIG` from `cmux identify --json` → all 4 panes equal 25%
- **4+ agents**: extend grid by splitting largest agent pane

**Files changed**: SKILL.md, orchestration.md, README.md, plugin.json, marketplace.json (92 insertions, 34 deletions + version bump to 1.6.0)

## Interactive Mode Fix (v1.9.0)

**Problem**: `claude -p` (print mode) buffers all output and only displays it when the agent finishes. Agent panes appeared stuck with a blinking cursor for minutes — no streaming tokens, no tool-call indicators, no visibility into progress. Users need to see agents working in real-time.

**Root cause**: The v1.6.0 change introduced `-p` to make agents auto-exit, but `-p` also suppresses all interactive output. The trade-off (auto-exit vs visibility) was wrong — visibility is more important since the orchestrator closes panes via `close-surface` anyway.

**Fix — Switch to interactive mode**:
1. Changed all `claude -p 'prompt'` → `claude 'prompt'` and `claude -p "$(cat file)"` → `claude "$(cat file)"` across SKILL.md, orchestration.md, README.md
2. `SKILL.md` "Launching Agents" — Rewritten: "Always launch agents in interactive mode". Explains that interactive mode shows real-time streaming, and agents don't auto-exit but orchestrator closes panes via `close-surface`
3. `SKILL.md` "Common Mistakes" — Replaced old `-p` entry with: "Using `claude -p` for spawned agents → Use interactive mode for streaming output"
4. `orchestration.md` — Same changes across all sections (launch, sync, retry, workspace)

**Why this is safe**: The original v1.6.0 concerns about removing `-p`:
- Workspace trust dialog → solved by pre-configuring permissions in `.claude/settings.json`
- Agent doesn't auto-exit → fine: orchestrator closes panes via `close-surface`
- Claude "improvising" with temp files → was about not knowing the CLI, not about `-p` itself

**Files changed**: SKILL.md, orchestration.md, README.md, plugin.json, marketplace.json (version bump to 1.9.0)

## Prompt File Fix (v1.8.0 → v1.8.1)

**Problem 1 (v1.8.0)**: Multi-line or complex prompts sent inline via `cmux send` get corrupted in agent panes. `cmux send` interprets `\n` as Enter, which splits multi-line prompts across shell lines — the shell sees an unclosed single quote and enters `quote>` continuation mode, often getting stuck or producing garbled input. This was especially common with detailed research prompts containing URLs, lists, and multi-paragraph instructions.

**Root cause**: `cmux send --surface $S1 "claude -p 'long\nmulti-line\nprompt'\n"` — each `\n` inside the prompt is converted to Enter by cmux, breaking the shell's quote parsing.

**Problem 2 (v1.8.1)**: The v1.8.0 fix used `cat file | claude -p` (piping), but `claude -p` requires the prompt as a command-line argument — it does NOT read from stdin. This caused agent panes to hang with a blinking cursor after displaying the command.

**Fix — File-based prompt with `$(cat)` command substitution**:
1. `SKILL.md` "Launching Agents" — Split into two sub-sections: "Simple prompts" (inline, unchanged) and "Complex or multi-line prompts" (write prompt to `scratchpad/agent-N-prompt.md`, then use `$(cat)` substitution)
2. `orchestration.md` "Launch Agents with Labels" — Updated primary examples to use the `$(cat)` pattern, with inline as fallback for simple one-liners
3. `SKILL.md` "Common Mistakes" — Added entries for inline multi-line prompts, piping to `claude -p`, and `/tmp/` usage

**Key pattern**:
```bash
# 1. Write prompt to file (orchestrator uses Write tool)
# 2. Use $(cat) to pass file content as argument:
cmux send --surface $S1 "claude -p \"\$(cat scratchpad/agent-1-prompt.md)\"\n"
```

**Why `$(cat)` not pipe**: `claude -p` expects a positional argument. `cat file | claude -p` sends content to stdin which `claude -p` ignores — it hangs waiting. `"$(cat file)"` expands the file content as the argument to `-p`.

**Files changed**: SKILL.md, orchestration.md, plugin.json, marketplace.json, README.md (v1.8.0 → v1.8.1)

## Agent Cleanup & Output Persistence (v1.7.0)

**Problem 1 — No per-agent cleanup**: Agent panes stayed open after finishing, cluttering the workspace. Cleanup was documented as a batch operation at the very end of orchestration, not per-agent as agents complete.

**Problem 2 — Ephemeral output**: Agent results were collected via `read-screen --scrollback` piped to `/tmp/` files or named buffers. Screen output is lost when panes close; `/tmp/` files are fragile and not project-local.

**Fix — Output persistence to `scratchpad/`**:
1. `SKILL.md` "Launching Agents" — Added bold callout: "Always instruct agents to save output to `scratchpad/`". Updated example prompts to include save instruction: `claude -p 'implement auth module. When done, save a summary of changes to scratchpad/agent-auth.md'`
2. `orchestration.md` "Launch Agents with Labels" — Same pattern applied to all 3 agent prompt examples
3. `orchestration.md` "Collect Results" — Rewritten to read from `scratchpad/` files as primary method, `read-screen` demoted to fallback

**Fix — Per-agent pane cleanup**:
1. `SKILL.md` "Cleanup" — Rewritten: close each agent's pane as soon as it finishes (not batch at end). Pattern: `test -f scratchpad/agent-auth.md && cmux close-surface --surface $S1`
2. `orchestration.md` "Update Progress and Clean Up" — Restructured to show per-agent cleanup with progress updates as each agent completes, sidebar cleanup after all finish
3. `SKILL.md` "Common Mistakes" — Added 2 entries: "Not persisting agent output" and "Leaving finished agent panes open"

**Files changed**: SKILL.md, orchestration.md, plugin.json, marketplace.json, README.md (version bump to 1.7.0)

## Skill Evaluation & Fixes (v1.3.0)

After research, we evaluated the claude-cmux-skill plugin against findings. 8 fixes were implemented:

| # | Fix | Files Changed |
|---|-----|---------------|
| 1 | **Output parsing** — `new-split` returns `OK surface:N workspace:N`, skill now documents parsing it | `SKILL.md` |
| 2 | **Permission handling** — new section on spawned agents hitting permission prompts, strategies to approve or pre-configure | `SKILL.md` |
| 4 | **Terminology** — "Subagent" → "Agent" across all headings and body text | `SKILL.md`, `plugin.json`, `orchestration.md` |
| 5 | **Lifecycle hooks** — SessionStart, Stop, Notification hooks forwarded to `cmux claude-hook` (gated on `$CMUX_SOCKET_PATH`) | `hooks/hooks.json` |
| 7 | **Version mismatches** — bumped to `1.3.0` across plugin.json, marketplace.json, README badge | `plugin.json`, `marketplace.json`, `README.md` |
| 8 | **README directory tree** — added `hooks/` directory | `README.md` |
| 9 | **Global CLI flags** — added `--socket`, `--window`, `--workspace`, `--surface` | `complete-cli.md` |
| 10 | **`respawn-pane`** — added as cleaner error recovery alternative | `orchestration.md` |

### Gaps NOT yet fixed (deferred)
- **3**: No task decomposition guidance (when to parallelize, file boundary rules)
- **6**: No git worktree pattern for isolation
- **5b**: No "when NOT to parallelize" breakeven analysis (tasks <15 min → multi-agent slower)

## Key Details

- **Decisions**:
  - cmux blocks the built-in Agent tool via PreToolUse hook when `CMUX_SOCKET_PATH` is set, redirecting to cmux panes for better isolation and observability.
  - Plugin ships SessionStart/Stop/Notification lifecycle hooks that delegate to `cmux claude-hook` for native sidebar status management.
  - Custom `set-status` in hooks was removed in v1.5.0 as redundant — `cmux claude-hook` handles lifecycle status natively.
  - Panes (`cmux new-split`) are the explicit default for agent orchestration. Workspaces (`cmux new-workspace`) are reserved for separate project roots only — documented in v1.5.1.
  - Spawned agents must use interactive mode (`claude 'prompt'`, NOT `claude -p`) for real-time streaming output — reversed from v1.6.0's `-p` recommendation in v1.9.0.
  - Pane splits must use `--surface` targeting with captured refs — documented in v1.6.0 to fix uneven recursive splitting.
  - Spawned agents must persist output to `scratchpad/` files — documented in v1.7.0 so results survive pane closure.
  - Agent panes should be closed per-agent as they finish (not batched at the end) — documented in v1.7.0 to reduce workspace clutter.
  - Complex/multi-line prompts must use file + `$(cat)` pattern (`claude "$(cat scratchpad/agent-N-prompt.md)"`) to avoid `cmux send` quoting corruption. Documented in v1.8.0, fixed in v1.8.1, updated to interactive mode in v1.9.0.
- **Open questions**: Should task decomposition rules and git worktree patterns be added to SKILL.md or orchestration.md?

## References

- `hooks/hooks.json` — PreToolUse Agent blocking + SessionStart/Stop/Notification lifecycle hooks
- `skills/using-cmux/SKILL.md` — Core cmux skill with balanced pane layouts, interactive mode invocation, scratchpad output persistence, per-agent cleanup, common mistakes
- `skills/using-cmux/references/orchestration.md` — Multi-agent patterns with balanced splits, interactive mode, scratchpad output, per-agent cleanup, error recovery with `respawn-pane`
- `skills/using-cmux/references/browser-automation.md` — Full browser API reference
- `skills/using-cmux/references/notifications.md` — Notification systems comparison
- `skills/using-cmux/references/complete-cli.md` — Complete CLI catalog (220+ commands), global flags
- `.claude-plugin/plugin.json` — Plugin manifest v1.9.0
- `.claude-plugin/marketplace.json` — Marketplace catalog v1.9.0
- `README.md` — Project README with interactive mode examples, version badge v1.9.0
