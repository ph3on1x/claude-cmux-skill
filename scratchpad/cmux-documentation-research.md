# cmux Documentation & Research

> Last updated by /write on 2026-03-24
> Source: Research session (3 parallel agents) + skill evaluation & fixes (v1.3.0) + hooks simplification (v1.5.0)

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

### Plugin Structure (claude-cmux-skill v1.5.0)
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
cmux send --surface surface:N "claude 'retry task'\n"  # Restart
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

- **Decisions**: cmux blocks the built-in Agent tool via PreToolUse hook when `CMUX_SOCKET_PATH` is set, redirecting to cmux panes for better isolation and observability. Plugin ships SessionStart/Stop/Notification lifecycle hooks that delegate to `cmux claude-hook` for native sidebar status management. Custom `set-status` in hooks was removed in v1.5.0 as redundant — `cmux claude-hook` handles lifecycle status natively.
- **Open questions**: Should task decomposition rules and git worktree patterns be added to SKILL.md or orchestration.md?

## References

- `hooks/hooks.json` — PreToolUse Agent blocking + SessionStart/Stop/Notification lifecycle hooks (simple `cmux claude-hook` delegation)
- `skills/using-cmux/SKILL.md` — Core cmux skill with CLI reference, orchestration patterns, permission handling, common mistakes
- `skills/using-cmux/references/orchestration.md` — Advanced multi-agent patterns, error recovery with `respawn-pane`
- `skills/using-cmux/references/browser-automation.md` — Full browser API reference
- `skills/using-cmux/references/notifications.md` — Notification systems comparison
- `skills/using-cmux/references/complete-cli.md` — Complete CLI catalog (220+ commands), global flags
- `.claude-plugin/plugin.json` — Plugin manifest v1.5.0
- `.claude-plugin/marketplace.json` — Marketplace catalog v1.5.0
- `README.md` — Project README with hooks directory tree
