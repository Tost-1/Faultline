# Faultline

Faultline is a macOS menu bar app. It watches crash, panic and hang reports, uses headless Claude Code to diagnose the ones that matter, and shows what happened and how to fix it.

**Conventions:** the same as Readout:
- YAGNI, DRY, SOLID. No code comments. Don't refactor outside the task.
- Commits are one plain imperative sentence, with no attribution, and the privacy hooks must pass.
- SwiftPM only, macOS 26, Swift 6, Swift Testing, no third-party dependencies, ad-hoc signed.
- No `foundry.yaml`.

**Package layout**
```
Package.swift            library FaultlineKit + executable Faultline + test target FaultlineKitTests
Sources/FaultlineKit/    watcher, parser, policy, diagnoser, store, public FaultlineView
Sources/Faultline/       App.swift, Panel.swift (same pattern as Uplink's)
Resources/Info.plist     LSUIElement, bundle id dev.faultline.Faultline, LSMinimumSystemVersion 26.0
scripts/build-app.sh     build → build/Faultline.app → ad-hoc sign → ~/Applications → open
```

**App shell:** the same as Uplink: AppDelegate, `NSStatusItem`, and a glass `NSPanel` hosting `FaultlineView`. The right-click menu has Settings…, Launch at Login and Quit.
- The status symbol shows a dot when there are unread diagnoses.
- It uses a distinct symbol while a diagnosis is running.

## Data layer (`Sources/FaultlineKit`)

**1. Watcher**
- Watch `~/Library/Logs/DiagnosticReports` and `/Library/Logs/DiagnosticReports` with an FSEvents stream. Both are readable without sudo; the system one works through `_analyticsusers` group membership.
- Debounce for 2 s, because files are written in stages, and wait until the size is stable.
- **On every launch,** scan for reports newer than the last-seen marker. This catches kernel panics written during the reboot before the app started.
- **On first launch,** record every existing report as seen and don't diagnose the backlog.
- The seen set is persisted.

**2. Parser and classification**

`.ips` files are two JSON documents: line 1 is the header, the rest is the body.
- **Header:** `bug_type`, `app_name`, `bundleID`, `is_first_party`, `os_version`, `timestamp`, `incident_id`.
- **Body:** `procName`, `procPath`, `exception`, `termination`, `faultingThread`, `threads`, `usedImages`, `asi`, `lastExceptionBacktrace`.

Kinds:

| Signal | Kind |
|---|---|
| `bug_type` 309 / `.crash` | crash |
| 210 / `.panic` / `panic-full-*` | kernel panic |
| 288 / `.hang` / `.spin` | hang |
| 298 (`JetsamEvent`) | memory pressure |
| 409 / `*watchdog*` | watchdog |
| `.diag` | resource |
| `.shutdownStall` | shutdown stall |
| anything else | other |

Build a compact **digest** per report, a few KB, so the prompt stays small:
- process, version, OS, exception type and codes, termination reason, `asi` messages
- the faulting thread's top ~25 frames as `image` + `symbol` + offset
- non-Apple images in the crashed stack

The Phase 0 spike decides the exact fields for panics and hangs.

**3. Policy** (decides whether a report auto-diagnoses)
- **Auto by default:** kernel panic, watchdog, and crash.
- **Listed only by default:** hang, memory pressure, resource, shutdown stall, other. Each kind has a toggle in Settings.
- **Never auto-diagnosed:** Faultline itself, `claude`, and Claude Code processes. This prevents loops.
- An ignore list of process names, added from the detail view with "Ignore this process".
- **Dedupe:** the same process plus the same exception type within 24 h links to the existing diagnosis instead of running again.
- **Daily cap:** auto-diagnoses per day default to 10. Manual runs are uncapped.

**4. Diagnoser**
- **Finding `claude`:** a GUI app doesn't inherit PATH. Copy the lookup order from the sibling QuickClaude app's `Sources/QuickClaude/Chat.swift` (the kickoff prompt gives its local path):
  1. `~/.local/bin`, `/opt/homebrew/bin` and `/usr/local/bin`
  2. the Claude Desktop bundled copy
  3. `zsh -lc 'command -v claude'`
- **Run:**
  ```
  claude -p <prompt> --model <setting, default sonnet> --output-format json --max-turns 10
  ```
  - Restrict tools to read-only: `Read`, `Grep`, `Glob`, and `Bash` limited to `log show:*`, `sw_vers`, `system_profiler:*` and `kextstat`. Nothing that writes.
  - Run with `cwd` set to `~/Library/Application Support/Faultline/work`.
  - Time out after 5 min.
  - Run one diagnosis at a time, from a FIFO queue.
  - **Phase 0 confirms the exact flag names on the installed version.** Check for `--allowedTools`, `--json-schema` (structured output), a no-session-persistence flag (so diagnoses don't clutter `~/.claude/projects` or Readout's counts), and a bare or no-hooks mode (so user hooks and plugins don't fire).
- **Prompt:** the digest, plus the report's file path so Claude can `Read` more if it needs to, and the macOS version. Ask for JSON with:
  - `headline` (one line)
  - `what_happened` (2–4 sentences, plain English)
  - `likely_cause`
  - `confidence` (low / medium / high)
  - `fix_steps` (ordered list of strings)
  - `needs_attention` (bool; false for one-off, harmless crashes)
- Parse it with `JSONDecoder`. If parsing fails, store the raw text as `what_happened`.

**5. Store:** `~/Library/Application Support/Faultline/incidents.json`. Each incident has an id, report path, kind, process, time, digest, status (new / queued / diagnosing / done / skipped(reason) / failed), the diagnosis and a read flag. Keep the last 200.

**6. Notifications** (`UNUserNotificationCenter`)
- "Faultline: <process> crashed. Diagnosing…" when a report is detected.
- "<headline>" when the diagnosis is done.
- Clicking either opens the panel on that incident.
- Phase 0 confirms notifications work for an ad-hoc-signed app in `~/Applications`.

## UI (public `FaultlineView`)
- **List:** newest first. Each row has a kind icon, the process, a relative time, and a status pill or headline. A filter shows all or diagnosed only.
- **Detail:**
  - headline
  - What happened
  - Likely cause, with confidence
  - numbered fix steps
  - buttons: Open Report (`NSWorkspace`), Copy Diagnosis, Re-diagnose, Diagnose (for skipped items), Ignore This Process
- **Settings:** auto-diagnose toggles per kind, the daily cap, the model, and the ignore list.

## Phases (each ends green: `swift build`, `swift test`, and a commit)
- **Phase 0: spike (gate).** Findings go to `docs/phase0.md`, scrubbed.
  1. Pick one real crash `.ips`, build the digest by hand and run `claude -p` with the planned flags. Confirm the flag names, structured output, the read-only enforcement (a write attempt is denied), the cost and duration, and whether a transcript is written.
  2. Confirm FSEvents fires for both directories, and that a notification posts from an ad-hoc-signed bundle.
  3. To get a real crash: `scripts/crash-test.sh` builds a tiny Swift binary that calls `fatalError()`. A panic can't be triggered safely, so panic parsing is tested only on fixtures.
- **Phase 1: shell.** Package, status item, empty panel, `scripts/build-app.sh`.
- **Phase 2: watcher, parser and store.** Tests on synthetic `.ips`, `.panic` and `.diag` fixtures cover classification, digest contents, the first-launch baseline and launch catch-up.
- **Phase 3: policy and diagnoser.** Tests cover the policy matrix, dedupe, the cap, self-exclusion and output parsing (good JSON, bad JSON). The runner is behind a protocol so tests don't call the real `claude`.
- **Phase 4: UI and notifications.**
- **Phase 5: README.** What it watches, what it sends (the digest goes to Claude through your Claude Code login), and settings.

## Verification
- `swift test` passes.
- After `scripts/build-app.sh`, running `scripts/crash-test.sh` produces this sequence within about 2 minutes: a notification, then a diagnosis, then a detail view with fix steps.
- Crashing it a second time links to the existing diagnosis instead of running again.
- Relaunching the app does not re-diagnose old reports.
- An ignored process stays listed and is never diagnosed.
- No new files appear under the user's `~/.claude/projects` from Faultline runs, if the no-persistence flag exists.
