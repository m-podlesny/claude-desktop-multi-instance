---
name: add-claude-desktop-instance
description: Add another Claude desktop app instance on Windows - its own login, MCP connectors, settings, skills and transcripts - by writing a .bat launcher that sets --user-data-dir and CLAUDE_CONFIG_DIR. Also use when explaining to a developer how the two directories differ, why a second instance sees different skills, or how config is shared between instances.
argument-hint: <instance-name>
---

<!-- allowed-tools: Bash, PowerShell -->

This is about the **Claude desktop app** - the Electron GUI. Not the Claude Code CLI, which is a
different program that also ships a `claude.exe` and takes none of these settings. See step 2.

**Windows only.** The paths and the `.bat` launcher are Windows-specific. macOS and Linux keep the
same two settings in different locations - none of that is verified here.

**Assumes a Squirrel install** - the ordinary download from claude.ai, which lands in
`%LOCALAPPDATA%\AnthropicClaude\`. Not the Microsoft Store (MSIX) build. Confirm with step 2 before
going further; if `Get-AppxPackage *Claude*` returns a package, stop and see the Provenance note.

That assumption is what makes this approach a good one. The launcher points at Squirrel's stub,
which resolves the newest `app-<version>` itself, so **every instance keeps auto-updating** and the
launcher never needs touching again. No copy of the app, nothing to re-copy after each release, and
no administrator rights. Community tools that tell you to copy Claude into a portable folder are
solving the MSIX problem, which a Squirrel install does not have - following them here would trade
a self-updating setup for one you have to maintain by hand.

**Instance name:** $ARGUMENTS

## 1 - The model

An instance is pinned by **two independent settings**. Nearly every confusing symptom in this area
comes from conflating them, so establish this before touching anything.

| | `--user-data-dir` | `CLAUDE_CONFIG_DIR` |
| --- | --- | --- |
| kind | command-line flag | environment variable |
| points at | a folder under `%APPDATA%` | a folder under `%USERPROFILE%` |
| what it is | a Chromium browser profile - the app is Electron | the agent's config folder, same layout the terminal CLI uses |
| decides your account | **yes**, the session cookie lives here | no |
| MCP connectors | yes | no |
| settings, `CLAUDE.md`, skills | no | **yes** |
| transcripts, memory | no | **yes** |
| safe to delete | yes, you sign in again | no, you lose history |

Set only the flag and you get a second login sharing one config root - same settings, same skills,
both accounts' transcripts in one `projects/`. Set only the variable and you get separate settings
under one login. Most people want both.

Electron keys its single-instance lock on `--user-data-dir`, so two instances pointed at two
folders run side by side as separate process trees, each one main process plus ~11 children.

### Signing in as a different account changes none of this

Every default path is derived from the **Windows user**, never from the Claude account:

| Default | Derived from |
| --- | --- |
| `~\.claude` | the home directory |
| `%APPDATA%\Claude` | Electron's `%APPDATA%\<productName>` |
| `~\Claude` | `app.getPath("home")` + `"Claude"` - Cowork user files |

Sign out and back in as someone else and all three still point at the same folders, so two accounts
share one config root, one login profile and one Cowork folder. **Overriding the paths is the only
thing that separates instances - the account signed in never does.** Say this explicitly; users
reasonably expect switching accounts to switch their data, and it does not.

### `skills\` must be a real directory

Never point a config root's `skills\` at a shared folder through an NTFS junction or a symlink. The
skill loader ignores reparse points and finds **zero** personal skills. Share skills with git
instead - see step 7, which explains why the symptom is so misleading.

### The third location: Cowork user files

Beyond the two settings above there is a third per-instance path, `coworkUserFilesPath`, stored in
`<app profile>\claude_desktop_config.json` and shown in Settings > Cowork > Cowork files. Resolution
order on a fresh install, from the app's own resolver:

1. `%USERPROFILE%\Claude` - the current default, used when local and verifiable.
2. `%USERPROFILE%\Documents\Claude` - the older default, used instead if it already exists with content.
3. `<app profile>\cowork-user-files` - fallback only when neither can be verified, e.g. a UNC home.

Only the third is per-instance by construction. **The default is not instance-aware**, so a second
instance left alone will share one Cowork folder with the first. Set it explicitly per instance -
`Documents\Claude\<instance>` keeps the old default's shape while staying separate.

**Leave the primary instance on the default and override only the secondary.** That way one
instance stays on a path the app would have chosen anyway, and the divergence is confined to the
instance that is already unusual. A worked example:

| Instance | Cowork folder | Classification |
| --- | --- | --- |
| primary | `%USERPROFILE%\Claude` | `home` - the stock default |
| secondary | `%USERPROFILE%\Claude<Instance>` | `custom` |

Name the secondary as a visible sibling of the default (`ClaudeWork` beside `Claude`) so the two
read as a pair in a directory listing. Avoid burying it inside `~\.claude*` - those are agent config
roots, and user documents do not belong in them.

**Setting it.** Two ways, and the file edit is the better one for this task.

Settings > Cowork > Cowork files changes one instance at a time, by hand, in a UI you have to be
signed into. Editing `<app profile>\claude_desktop_config.json` is scriptable, does every instance
in one pass, and leaves a diff you can check.

**The edit is safe while the app is running.** Verified 2026-09-15: both instances were open when
their files were rewritten, and both kept the new value across a restart. The app reads
`coworkUserFilesPath` at startup and caches it, but does not write the file back on quit, so there
is no clobber to race. You do not need to close anything first.

Do it with these five precautions:

- back the file up first - it holds unrelated keys (`preferences`, `globalShortcut`);
- replace only the one value with a literal string replace, never by re-serialising the JSON, which
  would reformat the whole file;
- re-parse the result before writing, to catch a malformed edit;
- create the target directory first, so the stored path verifies instead of falling back to
  `<app profile>\cowork-user-files`;
- restart the instance and re-read the file. The new path is live only after a restart, because the
  running instance is still using the one it cached at startup.

The app `mkdir`s the resolved path at startup with mode 0700 - which is why a Cowork folder you
delete keeps coming back until you change the setting pointing at it.

`reference/two-instances-anatomy.md` is a filled-in example of a finished setup. Reach for it when
the user wants to understand the arrangement rather than build one.

## 2 - Locate the executable

```powershell
Test-Path "$env:LOCALAPPDATA\AnthropicClaude\claude.exe"
```

That root `claude.exe` is a ~363 KB **stub**. The real ~246 MB binaries sit in `app-<version>\`
beside it, alongside `Update.exe` - the Squirrel signature. **Point the launcher at the stub.** It
resolves the newest `app-<version>` itself and forwards its command line verbatim, so the launcher
survives auto-updates. Machines routinely hold two `app-<version>` folders at once, so a hardcoded
one is wrong the moment the app updates.

**Three different things are named `claude.exe`.** Disambiguate by `ProductName`:

```powershell
(Get-Item <path>).VersionInfo | Format-List ProductName, ProductVersion
```

| path | size | ProductName |
| --- | --- | --- |
| `%LOCALAPPDATA%\AnthropicClaude\claude.exe` | ~363 KB | Claude (the stub - use this) |
| `%LOCALAPPDATA%\AnthropicClaude\app-<version>\claude.exe` | ~246 MB | Claude (real app - do not hardcode) |
| `%USERPROFILE%\.local\bin\claude.exe` | ~227 MB | Claude Code (the CLI - wrong program) |

The desktop app is **not** on PATH and registers no `App Paths` key under HKCU or HKLM, so there
is no short name for it - a bare `claude` resolves to the CLI, which takes no `--user-data-dir`.

## 3 - Pick the paths

| Thing | Convention |
| --- | --- |
| Script | `claude-<instance>.bat` in a directory already on PATH |
| App profile | `%APPDATA%\Claude-<instance>` |
| Config root | `%USERPROFILE%\.claude-<instance>` |

**The app profile's parent is constrained; its leaf name is not.** It must sit directly under
`%APPDATA%`, beside the stock `%APPDATA%\Claude` (confirm that one holds `Local State`,
`Local Storage`, `IndexedDB`). `Claude-<instance>` sorts next to it; `.claude-<instance>` sorts to
the top. A leading dot is legal on NTFS and confers no hidden attribute - that is a Unix
convention, not a Windows one. Nothing in the app depends on the name, so ask rather than assume,
and if you rename something the user chose, say plainly that it is taste and not a requirement.

**The config root has no constraint.** Flat at `%USERPROFILE%\.claude-<instance>`, beside the
default `~\.claude`. Do not nest it under an extra folder unless asked.

```powershell
[Environment]::GetEnvironmentVariable('Path','User') -split ';' | Where-Object { $_ }
```

`%USERPROFILE%\.local\bin` is the usual personal bin. Only edit PATH if no such directory exists.
**Never name the script `claude.bat`** - it would shadow the CLI.

## 4 - Write the launcher

```bat
@echo off
set "CLAUDE_CONFIG_DIR=%USERPROFILE%\.claude-<instance>"
start "" "%LOCALAPPDATA%\AnthropicClaude\claude.exe" --user-data-dir="%APPDATA%\Claude-<instance>"
```

Each line is load-bearing:

- **`set` before `start`** - the launched process inherits the cmd environment, which is how the
  variable reaches the app. No user-level env var is needed, and one would leak into the default
  instance too.
- **`start ""`** - the empty first argument is the window title. Without it, `start` treats the
  quoted exe path as a title and launches nothing.
- **`start` at all** - detaches the app so the console closes instead of blocking.
- **`%VARS%`, not literal paths** - cmd expands them at run time, keeping the script portable.

Write it with CRLF endings. From Bash use a quoted heredoc (`<<'EOF'`), never `printf` - `printf`
interprets escapes in a path like `\backend-agent` (`\b` -> backspace) and silently corrupts it.

## 5 - Seed the config root

A new config root starts empty: no settings, no skills, signed out.

**If the user keeps a config repo** (`git -C %USERPROFILE%\.claude remote -v`), clone it. That is
also the sharing mechanism - see step 7.

**Otherwise copy the existing root, then delete `.credentials.json` from the copy.** Leaving it
puts the first account's token inside the second account's config root.

## 6 - Verify

Statically:

```powershell
Get-Command claude-<instance> | Format-List Name, CommandType, Source
Get-Content <script> | Select-Object -Skip 1 | ForEach-Object { [Environment]::ExpandEnvironmentVariables($_) }
```

`Source` must be the script just written, and the expanded lines must name the stub, not the CLI.
**Test-Path both expanded paths.** A misspelled profile dir does not error - the app silently
creates a fresh blank signed-out one, which looks like a lost login rather than a typo.

Behaviourally, after the first launch:

```powershell
# which app profile each running instance uses
Get-CimInstance Win32_Process -Filter "Name='claude.exe'" |
  Where-Object { $_.CommandLine -match 'user-data-dir' } |
  ForEach-Object { $_.CommandLine -replace '^.*--user-data-dir=', '' } | Sort-Object -Unique
```

- `%APPDATA%\Claude-<instance>` now exists and holds `Local State` - the flag took.
- A new session appears under `<config-root>\projects\` rather than `~\.claude\projects\` - the
  variable took. Inside that instance's own Code tab, `$env:CLAUDE_CONFIG_DIR` shows it directly.

## 7 - Sharing config between instances

**Use git. Nothing else works reliably.**

Make each config root a clone of one private repo. A skill or settings change then travels by
commit + push in one instance, `git pull` in the other. Restart the receiving instance to load it.

The repo should deny everything by default and allow only config, so per-account state stays local:

```gitignore
*
!.gitignore
!CLAUDE.md
!settings.json
!mcp.json
!skills/
!skills/**
```

That leaves `projects/`, `history.jsonl`, `.credentials.json` and the caches untracked - each
instance keeps its own transcripts and its own login.

**Do not use junctions or symlinks.** Verified 2026-09-15: pointing a root's `skills\` at a shared
folder through an NTFS junction makes the skill loader find **zero** personal skills - only the
bundled Anthropic ones survive. It is easy to misdiagnose, because a running instance keeps
whatever it loaded before the swap; the breakage only appears on the next cold start, by which
point it looks unrelated. `skills\` must be a real directory in every config root.

## Failure modes

| Symptom | Cause |
| --- | --- |
| New instance is signed out and empty | Expected on first run - or the launcher points at a path that does not exist, and the app made a blank one |
| Instance sees none of the user's skills | `skills\` is a junction or symlink; it must be a real directory |
| Both instances share transcripts and settings | Only `--user-data-dir` was set; `CLAUDE_CONFIG_DIR` is missing |
| Both instances share a login | Only `CLAUDE_CONFIG_DIR` was set; `--user-data-dir` is missing |
| Both instances share one Cowork folder | `coworkUserFilesPath` left at its default, which derives from the Windows user and is not instance-aware |
| Launcher opens the CLI, not the app | The script calls a bare `claude`, which resolves to `.local\bin\claude.exe` |
| Launcher stopped working after an update | It hardcoded `app-<version>\claude.exe` instead of the stub |
| Second account's root holds the first account's token | The root was seeded by copying without deleting `.credentials.json` |

**Moving an existing profile or config root** means moving the directory, not just editing the
path. Check whether the old one exists first; if it does, move it and keep the login.

**Removal:** delete the `.bat`, then the app profile under `%APPDATA%` and the config root.
Deleting only the script leaves both directories on disk.

## Provenance

Verified first-hand on a Squirrel install, 2026-09-15, on a live two-instance setup: the three
`claude.exe` binaries and their sizes and `ProductName` values; two `app-<version>` folders present
simultaneously; the desktop app absent from PATH and from `App Paths` in both HKCU and HKLM; `set`
in the `.bat` reaching the app; a junctioned `skills\` yielding zero personal skills on a cold
start while a running instance kept its own; two config roots cloned from one private repo sharing
skills correctly; a launcher pointing at a misspelled profile dir failing silently. The Cowork
resolution order, the three candidate paths and the 0700 mkdir were read directly out of
`resources\app.asar` (`resolveCoworkUserFilesPath`) in app-1.52386.6, and confirmed against two live
instances - one sitting on the `home` default, one on a stale `custom` path. Editing
`claude_desktop_config.json` under a running app was then confirmed end to end: both instances were
open during the rewrite, both were restarted, and both held the new path afterwards.

**Not verified here, and out of scope:** MSIX installs (`Get-AppxPackage *Claude*` returned nothing on this machine). If one turns up, say so rather than improvising - the whole stub-and-auto-update argument above does not hold, and the workaround is a different design.
If one is found, Windows blocks launching that `.exe` directly - either re-resolve `InstallLocation`
at each launch or copy the app to a normal folder. Also unverified: the claim that the Cowork
Hyper-V workspace feature requires the app profile to sit under `%APPDATA%` because it looks for
`rootfs.vhdx` there. Both come from community multi-instance tools - re-check before relying on them.
