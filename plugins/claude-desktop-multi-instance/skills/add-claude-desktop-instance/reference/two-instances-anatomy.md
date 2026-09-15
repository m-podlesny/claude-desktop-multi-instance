# Anatomy of two Claude Desktop instances

A worked example of a finished two-instance setup on Windows. Use it as the target shape when
building a new instance, and as the checklist of what must be decided for each one.

The example uses a primary instance left entirely on its defaults, and a second one called `work`.

| | Primary | Work |
|---|---|---|
| **How to start** | normal Start-menu shortcut | `claude-work` — a `.bat` on PATH (`%USERPROFILE%\.local\bin\claude-work.bat`), typed in any terminal |
| **Logged in as** | first account | second account |
| **`--user-data-dir`** *(command-line flag)* | `%APPDATA%\Claude` *(default, not passed)* | `%APPDATA%\Claude-work` *(set by the .bat)* |
| ↳ what that folder is | a Chromium browser profile — the desktop app is Electron, so it keeps one. Holds the login cookie, **not** just cache | same kind of folder, its own copy |
| ↳ controls | session cookie (= which account), device identity, MCP connector definitions and their tool toggles, window size/position, browser cache | its own separate set of all the same things — different account, different connectors, different window state |
| **`CLAUDE_CONFIG_DIR`** *(environment variable)* | `%USERPROFILE%\.claude` *(default, unset)* | `%USERPROFILE%\.claude-work` *(set by the .bat)* |
| ↳ what that folder is | the agent's config folder — same layout the terminal CLI uses, nothing Electron about it | same kind of folder, its own copy |
| ↳ controls | `settings.json` (permissions, hooks, theme, effort), `CLAUDE.md`, keybindings, `skills\`, `plugins\`, `projects\` (transcripts + memory), prompt history | its own separate copy of each — though settings, `CLAUDE.md` and skills can be kept in step by cloning both roots from one repo |
| **`coworkUserFilesPath`** *(app-profile setting)* | `%USERPROFILE%\Claude` *(the stock default)* | `%USERPROFILE%\ClaudeWork` *(custom)* |
| ↳ what that folder is | Cowork user files. Stored in `<app profile>\claude_desktop_config.json`, shown in Settings → Cowork → Cowork files | same kind of folder, its own copy |
| ↳ why it is set by hand | the default derives from the Windows user, so it is **not** instance-aware — left alone, both instances would share one folder | overridden precisely to avoid that collision |
| **Skills** | `%USERPROFILE%\.claude\skills\` — real folder | `%USERPROFILE%\.claude-work\skills\` — real folder |
| **Git** | clone of a private config repo, on `main` | clone of the same repo, on `main` |
| ↳ what's tracked | config only — `CLAUDE.md`, `settings.json`, `mcp.json`, `skills\`. A deny-all `.gitignore` with an allowlist, so transcripts, history and credentials stay local | same; this is how a skill gets from one instance to the other — commit and push in one, `git pull` in the other |
| **Transcripts / memory** | `%USERPROFILE%\.claude\projects\` — separate | `%USERPROFILE%\.claude-work\projects\` — separate |
| **Processes** | 12: 1 main + 11 children | 12: 1 main + 11 children |
| ↳ what a "tree" is | one main process spawns the rest — GPU, network service, one renderer per window, utilities | its own independent tree — no shared PIDs, separate folders held open |

## Where the rules live

This file is the worked example only. Every rule behind it — the two settings, the Cowork
resolution order, why `skills\` must be a real directory, and why config is shared by git — is in
`SKILL.md`. Do not restate them here; they will drift.

## Checking a running instance

Which app profile each running instance uses:

```powershell
Get-CimInstance Win32_Process -Filter "Name='claude.exe'" |
  Where-Object { $_.CommandLine -match 'user-data-dir' } |
  ForEach-Object { $_.CommandLine -replace '^.*--user-data-dir=', '' } | Sort-Object -Unique
```

Which config root — run inside the instance's own Code tab:

```powershell
$env:CLAUDE_CONFIG_DIR
```

Empty means the default `%USERPROFILE%\.claude`. To confirm behaviour rather than intent, compare
the newest file timestamp under each root's `projects\`.
