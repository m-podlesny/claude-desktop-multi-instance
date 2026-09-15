# claude-desktop-multi-instance

Run multiple Claude Desktop instances on Windows, each with its own account, MCP connectors,
settings, skills and transcripts. Packaged as a Claude Code skill.

## Install

```
/plugin marketplace add m-podlesny/claude-desktop-multi-instance
```

Then invoke `/claude-desktop-instance` and say what you want the new instance called.

## What it does

An instance is pinned by two independent settings, and most confusion here comes from
conflating them:

| | `--user-data-dir` | `CLAUDE_CONFIG_DIR` |
|---|---|---|
| kind | command-line flag | environment variable |
| controls | login, MCP connectors, window state | settings, `CLAUDE.md`, skills, transcripts |

Set only the first and you get a second login still sharing one config root. Set only the
second and you get separate settings under one login. The skill sets both, from a `.bat`
on your PATH.

It also covers a third per-instance path — Cowork user files — whose default is derived
from the Windows user and so is *not* instance-aware.

## Scope

- The **Claude desktop app**, not the Claude Code CLI.
- **Windows** only.
- Assumes a **Squirrel** install (the ordinary download from claude.ai), not the Microsoft
  Store MSIX build. The launcher points at Squirrel's stub, so every instance keeps
  auto-updating — no portable copy, nothing to re-copy after a release, no admin rights.

## Notes

- Share skills between instances with git, not junctions or symlinks. A junctioned
  `skills\` makes the loader find zero personal skills, and the breakage only appears on
  the next cold start.
- Signing in as a different account changes none of the default paths. They all derive
  from the Windows user.

## License

MIT
