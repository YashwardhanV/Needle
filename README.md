# Needle

A small always-on-top speedometer for Claude Code. The needle shows how much of the current phase is done,
taken from Claude's own task list (6 of 12 tasks done = 50%), and the colour tells you when Claude needs
permission, is waiting for you, looks stalled, or has finished the phase.

```
Claude Code event → hook/needle-hook.mjs → sessions/<id>.json → widget (file watcher) → gauge
```

The hook and the widget never talk to each other. Closing the widget cannot affect Claude Code, and Claude
Code can never reopen the widget.

## Install

Needs Node 18+. To build the widget: Rust and the [Tauri 2 prerequisites](https://tauri.app/start/prerequisites/).

```sh
node hook/install.mjs install --claude-md   # hooks into ~/.claude/settings.json (+ backup), snippet into ~/.claude/CLAUDE.md
node hook/install.mjs status
node hook/install.mjs uninstall             # removes only Needle's entries and snippet

cd app && npm install
npm run tauri dev      # run it
npm run tauri build    # Windows installer in app/src-tauri/target/release/bundle/nsis
```

Nothing starts automatically: no Startup entry, Run key, scheduled task or service. You launch the widget
when you want it. X or tray **Quit** ends the process.

## Using it

- Say the phase in your prompt ("build phase 2"), or prefix tasks with `[Phase 2]`.
- Total phases: add `needle.json` to the project root, either `{ "totalPhases": 5 }` or
  `{ "planFile": "PHASES.md" }` (counts `## Phase` headings).
- When tasks are added mid-phase, the needle moves back honestly and a "+N tasks" badge appears.
- Right-click: Always on top, Compact mode, Permission notifications (off by default), Quit.
- Several sessions show as dots. The most recent one is shown; any session that needs permission takes over.

Settings live in `%APPDATA%\Needle\config.json` (`stallMinutes`, default 5; `longRunningMinutes`, default 3).
State files live in `%LOCALAPPDATA%\Needle\sessions\`. Hook errors go to `%LOCALAPPDATA%\Needle\hook-errors.log`.
Set `NEEDLE_HOME` to move both.

## Token cost

The hooks themselves cost no tokens. The CLAUDE.md snippet is 3 sentences and asks Claude for one update per
finished task, with no `in_progress` or `TaskList` calls: 2 task calls per task instead of 3.
See [docs/phase0-findings.md](docs/phase0-findings.md).

## Download

Every push builds a Windows installer on GitHub Actions: open the latest **Build** run under the
repository's Actions tab and download the `needle-windows-installer` artifact.

## Development

```sh
node --test hook/test.mjs                 # hook + installer tests (replays real captured payloads)
cd app && npm run dev                     # browser preview with demo data:
# http://localhost:1420/?state=needs_permission&done=3&total=8   (also &compact, &added=2)
```

## Status

| Phase | State |
|---|---|
| 0 Discovery | Done. Real payloads for every event, including the permission prompt, in `docs/payloads/`. |
| 1 Hook + state files | Done. 10 tests; ~40 ms per hook run; verified in real Claude Code sessions. |
| 2–5 Widget | Done. Builds on Linux; launch, live file watching and single instance checked on a virtual display. CI builds the Windows installer. |

The Windows-only items in the acceptance checklist (Task Manager, Startup folder, idle RAM) still need one run on a real Windows PC.

## License

Copyright 2026 Yashwardhan Verma. Licensed under the [Apache License 2.0](LICENSE).
