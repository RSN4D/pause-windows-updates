# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository ships

Three hand-edited artifacts, no build step:

- `wputility.bat` — ~3000-line monolithic Windows batch utility (WPU). Self-elevates to admin via `powershell start -verb runas`. Requires Windows 10+ (aborts if `osMajor < 10`).
- `windows-updates-pause.reg` — standalone registry script that pauses updates until 2051-12-31.
- `windows-updates-unpause.reg` — the inverse.

There are no tests, no package manager, no compile step, no linter. Edits go straight to the shipped files. Releases are cut by manually dispatching `.github/workflows/release.yml` (workflow_dispatch only), which just uploads the three files as artifacts and creates a GitHub Release.

## Running / "testing"

The only way to validate changes is to run the `.bat` on a real (or VM) Windows 10/11 machine with admin rights. Double-click `wputility.bat` and walk through the menu paths your change touches. There is no headless test harness.

To verify pause state without the utility: `ms-settings:windowsupdate` (Settings → Windows Update) or `reg query "HKLM\SOFTWARE\Policies\Microsoft\Windows\WindowsUpdate\AU" /v NoAutoUpdate` (`0x1` = disabled).

## The three-way duplication that bites every change

The same registry writes appear in **all three** files:

- `:taskUpdatesDisable` in `wputility.bat` (~line 2509) mirrors `windows-updates-pause.reg`.
- `:taskUpdatesEnable` in `wputility.bat` (~line 2580) mirrors `windows-updates-unpause.reg`.

When you change a registry key in one of those locations (e.g. issue #18 added `ExcludeWUDriversInQualityUpdate`), the same change must land in the matching `.reg` file. They are not generated from a common source — drift is silent and only surfaces as a user bug report ("the .reg file doesn't disable as thoroughly as the .bat does").

The canonical pause dates are `2025-01-01T00:00:00Z` (start) and `2051-12-31T00:00:00Z` (end). These literals are duplicated across both `.reg` files and `:taskUpdatesDisable`. README copy ("12-31-2051") also references the end date.

## wputility.bat architecture

### Label naming is the architecture

Control flow is `goto :label` with conventional prefixes. The header comment (lines 10-42) lists them; match the prefix when adding new code:

- `menu*` — render an interactive menu and read a choice into `q_mnu_*`
- `task*` — perform a non-interactive action (registry writes, service control, etc.)
- `prompt*` — confirmation prompts called from menus before destructive `task*`
- `sess*` — terminal states: `sessQuit`, `sessFinish`, `sessError`, `sessAdvanced`
- `helper*` / `action*` / `forceQuit` — internal utilities

`:main` is the top-level menu. Anything user-facing eventually flows back to `:main` or one of the `sess*` exits.

### Data tables drive the menus

Lists are encoded as pseudo-arrays of pipe-delimited fields, then iterated with `for /f "tokens=... delims=[]|=" %%v in ('set arrayName[ 2^>nul')`. The field layout is documented in comments above each declaration — read those before editing. Examples:

- `set "servicesUpdates[wuauserv]=Windows Update Service|wuauserv"` — Windows Update service list (~line 218)
- `set "servicesUseless[NN]=Display Name|service_name"` — debloat services (~line 231); numeric index is what the user types in the menu
- `set "usersDisable[N]=Display|account_name"` — managed local users
- `set "schtasksDisable[N]=\path\to\task"` — scheduled tasks to disable
- `set "crapware[N]=Microsoft.PackageName"` — bloat packages; `crapwareIndexMax` (~line 274) must be updated when changing the range
- `set "apps[NN]=Display|PackageId|pkgManager"` — installable apps; `pkgManager` is `winget` or `powershell`
- `set "rTweaksGeneral[NN]=group_id|name|reg_path|reg_key|reg_type|val_enable|val_disable|is_secondary"` — registry tweaks (format documented at lines 319-343). Multiple rows sharing one `group_id` get applied together as a single user-facing tweak; only the first row's name shows in the menu, and secondary rows must set `is_secondary=true`.

Adding a service/app/tweak means appending one row to the table — the menu rendering loops pick it up automatically. No menu code changes needed unless the prefix/format is new.

### OS detection

Build number is read three ways and reconciled (~lines 433-501): `wmic`, `ver`, and `HKLM\...\CurrentVersion\CurrentBuildNumber`. If they disagree, the script warns and uses the backup value. Codename (`24H2`, etc.) is similarly resolved from registry first then mapped from build number as a fallback (~lines 522-582). If you add support for a new Windows release, both the codename map *and* the build-number ranges need updating.

### Side-effect locations

- Registry backups → `./registryBackup/` (gitignored, created by `:taskRegistryBackup`).
- winget package cache → `./cache/wputility.out` (used by `:menuAppsManage` to mark apps installed/uninstalled).
- `set "repo_version=1.6.0"` at line 117 is the version shown in the banner; bump it when cutting a release.

### Batch gotchas

- Most blocks use `setlocal enabledelayedexpansion`; refer to variables set inside `for`/`if` blocks with `!var!`, not `%var%`.
- ANSI color literals (`set "red=[38;5;160m"`, etc.) include a real ESC character — preserve it byte-for-byte when copy-editing.
- `echo:` is the idiom for a blank line (not `echo.`).
- After every `reg add` block that matters, the script checks `%errorlevel%` and jumps to `sessError` on failure — follow that pattern for new writes.

## Compliance with the user's global rules

The user's global CLAUDE.md forbids silently suppressing compiler warnings. This repo has no compiler, so the rule rarely applies — but the spirit ("don't paper over failures") translates to: don't replace a failing `reg add` with `2>nul` or strip the `errorlevel` check to make output look cleaner. Report the failure path.
