# claude-cmux-skill (now `cmux`) has moved

This project now lives in **[ph3on1x/agent-plugins-skills](https://github.com/ph3on1x/agent-plugins-skills/tree/main/skills/cmux)**, together with my other
agent skills and plugins. This repository is archived and no longer updated. The full git history and
release tags were carried over to the new repository.

## Install

Claude Code:

```text
/plugin marketplace add ph3on1x/agent-plugins-skills
/plugin install cmux@ph3on1x
```

Codex, Cursor, Gemini CLI, Antigravity and other agents:

```bash
npx skills add ph3on1x/agent-plugins-skills --skill cmux
```

## Already installed from this repository?

The plugin was renamed from `claude-cmux-skill` to `cmux`, because Claude Code reserves the `claude-` prefix for Anthropic. Reinstall once: `claude plugin uninstall claude-cmux-skill@claude-cmux-skill`, then the Claude Code commands above. The `/cmux` skill itself is unchanged.
