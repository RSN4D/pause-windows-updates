# Network / Remote-Connection Evaluation

This document lists every place in the **shipped, user-run artifacts** of this repository
(`wputility.bat`, `windows-updates-pause.reg`, `windows-updates-unpause.reg`) where code
reaches out — or *could* reach out — to the internet or a remote host on its own, what host
it contacts, and the likely purpose.

A separate section at the bottom covers the GitHub Actions workflows under `.github/`, which run
on GitHub's servers (not on the end-user's machine) and therefore are a different trust boundary.

**Top-line summary:** The utility makes **no automatic / silent outbound connections**. Nothing
phones home on launch, there is no update checker, and the two `.reg` files are 100% local
registry edits. The only code that actually talks to the internet is the **`winget` package
manager**, and it only runs when the user explicitly picks an install/uninstall/list action from
a menu. Everything else listed below is a URL that is merely *printed* on screen or sits in a
code comment.

---

## 1. `wputility.bat` — actual outbound connections (winget)

`winget` (the Windows Package Manager) contacts Microsoft's package source
(`cdn.winget.microsoft.com` / the `msstore` and `winget` sources) and then the individual
vendor download CDNs for whatever package is being installed. All of these are **user-triggered**
— they fire only after the user selects an app-management menu option and confirms; none run at
startup.

| Line | Command | Connects to | Purpose | Trigger |
|------|---------|-------------|---------|---------|
| 840 | `winget list > "%dir_cache%\%~n0.out" 2>&1` | Microsoft winget source | Builds the local app cache so the Apps Manager can show which apps are installed. Runs inside `:menuAppsManage`, behind a confirmation prompt. | User opens Apps Manager |
| 2157 | `winget list \| findstr /i %package%` | Microsoft winget source | Checks whether a package is already installed before installing. | `:taskAppsInstall` |
| 2161 | `winget install --id %package% --accept-source-agreements --accept-package-agreements --silent` | Microsoft + vendor CDN | Downloads and installs the chosen app. | `:taskAppsInstall` |
| 2163 | `winget install --id %package% --source %source% --accept-source-agreements --accept-package-agreements --silent` | Microsoft + vendor CDN | Same as above, but pinned to a specific source. | `:taskAppsInstall` |
| 2201 | `winget list \| findstr /i %package%` | Microsoft winget source | Checks install state before uninstalling. | `:taskAppsUninstall` |
| 2207 | `winget uninstall --id %package%` | (local, but `winget` may contact its source for metadata) | Removes the chosen app. | `:taskAppsUninstall` |
| 1912 | `call :taskAppsInstall winget 9NHT9RB2F4HD` | Microsoft + vendor CDN | Installs Microsoft Copilot (one of the menu actions). Routes through `:taskAppsInstall` above. | Menu action |
| 2042–2044 | `call :taskAppsUninstall winget <id>` | Microsoft winget source (metadata) | Uninstalls Copilot-related packages. Routes through `:taskAppsUninstall` above. | Menu action |

The catalog of installable apps that feed these commands is the `apps[NN]=...|<pkgId>|winget`
table at lines **386–426** (7-Zip, Chrome, Firefox, VS Code, Tor Browser, etc.). Selecting any of
them causes a winget download from Microsoft + the vendor's CDN.

> Note: `winget list` enumerates locally-installed packages but also queries the configured
> package source, so it can generate network traffic even when only "listing."

---

## 1b. `wputility.bat` — actual outbound connections (DISM)

`DISM /Online` operates on the running Windows image. Two of its modes can **reach out to
Windows Update / Microsoft servers** to download payload when the needed files are not already
present locally. Both are user-triggered menu actions, not automatic.

| Line | Command | May connect to | Purpose | Trigger |
|------|---------|----------------|---------|---------|
| 1601 | `dism /Online /Cleanup-Image /RestoreHealth /NoRestart` | Windows Update | Repairs the component store; if the local repair source is missing/corrupt, DISM downloads replacement files from Windows Update. | SFC/DISM "restore health" menu (`~line 1572`) |
| 2102 | `dism.exe /online /enable-feature /featurename:Recall /all /norestart` | Windows Update | Enables the "Recall" optional feature; DISM fetches the feature payload from Windows Update if it isn't staged locally. | Recall toggle menu |

For reference, the following DISM/Appx calls are **local-only (no network)**:

- Line 1780 `DISM /Online /Get-FeatureInfo ...` — reads feature state locally.
- Lines 2046–2047, 2197 `Get-AppxPackage ... | Remove-AppxPackage` — removes installed apps locally (this is how the **crapware** debloat list at lines 275+ and `:taskCrapwareUninstall` are uninstalled).
- Line 2153 `Get-AppXPackage ... Add-AppxPackage -Register ...\AppXManifest.xml` — re-registers an app from its existing local install folder.

---

## 2. `wputility.bat` — URLs that are displayed only (no connection)

These strings are printed to the console or stored in a variable. The script never opens or
fetches them; the user would have to copy/paste them into a browser manually.

| Line | URL | What it is |
|------|-----|-----------|
| 115 | `https://github.com/Aetherinox/pause-windows-updates` | Stored in `repo_url`; shown in the banner. |
| 635 | `%repo_url%` | Prints the repo URL in the header UI. |
| 741 | `https://buymeacoffee.com/aetherinox` | Donation link printed in the help/about text. |
| 1190–1191, 1273–1274 | `https://superuser.com/a/1152800`, `https://windowsreport.com/anniversary-update-defaultuser0` | Reference links shown to the user about the `defaultuser0` account cleanup. |

---

## 3. `wputility.bat` — URLs in code comments only (no connection)

Pure documentation references inside comment blocks; never executed.

| Line | URL | Context |
|------|-----|---------|
| 46 | `https://hahndorf.eu/blog/windowsfeatureviacmd` | `@ref` in the header comment about managing Windows features. |
| 1166–1167 | `https://superuser.com/a/1152800`, `https://windowsreport.com/...` | `@ref` comments for the user-account logic. |
| 2555, 2624 | `https://github.com/Aetherinox/pause-windows-updates/issues/18` | `@ref` comments tied to the driver-update tweak. |

---

## 4. What does *not* connect out

- **`powershell start -verb runas` (line 87) and the buffer-resize PowerShell call (line 105)** —
  these are local self-elevation / console-config calls. No network.
- **`actionProgUpdate` (line 42)** — listed in the header's label index but **not implemented or
  called anywhere**. There is no auto-update / version-check routine that contacts a server.
- **The `Get-AppxPackage` / `Remove-AppxPackage` / DISM examples in the header (lines 48–78)** —
  these are documentation in the header comment block. (The *executable* DISM calls that can hit
  Windows Update are covered separately in section 1b above; the Appx calls that do run are
  local-only.)
- **`windows-updates-pause.reg` and `windows-updates-unpause.reg`** — these are pure local
  registry edits. They contain `@url`/`@ref` comments pointing at `github.com` and
  `learn.microsoft.com` (e.g. pause.reg lines 6, 89, 134, 164; unpause.reg lines 5, 30, 54), but
  comments in a `.reg` file are inert. The keys they write *control how Windows itself talks to
  Microsoft Update* (e.g. `NoAutoUpdate`, `UseWUServer`, the `Pause*` timestamps), but the files
  perform no connection of their own.

---

## 5. GitHub Actions workflows (`.github/`) — runs on GitHub, not on the user's PC

These do not execute when a user runs the `.bat` or imports a `.reg`; they run in GitHub's CI
when the repo maintainer pushes, releases, or processes issues. They are included for
completeness because they *do* make remote calls.

| Workflow | Connects to | Purpose |
|----------|-------------|---------|
| `ping-developer.yml` | An SMTP mail server (`dawidd6/action-send-mail`, secrets-configured) | Emails the maintainer when someone uses a `/ping` comment. The email HTML also references remote images (e.g. an iconfinder CDN logo). |
| `issues-new.yml`, `issues-scan.yml`, `issues-stale.yml`, `issues-accept.yml`, `labels-create.yml`, `labels-clean.yml` | GitHub REST/GraphQL API (`api.github.com`) | Automated issue triage, labeling, and stale-issue management. |
| `release.yml` | GitHub API + release/changelog actions | Builds the release and uploads the three artifacts. |
| `cache-clean.yml`, `history-clean.yml`, `deploy-clean.yml`, `gpg-tests.yml` | GitHub API / GitHub infrastructure | Repo maintenance and CI tasks. |
| All workflows | GitHub Actions infra (`actions/checkout`, `actions/upload-artifact`, etc.) | Standard CI plumbing. |

---

## Bottom line for an end user

If you run `wputility.bat` and stay out of the **Apps Manager / install / uninstall** menus, the
utility does not initiate any internet connection on its own. Using those menus invokes `winget`,
which downloads software from Microsoft and the relevant vendor CDNs — expected behavior for a
package installer. The two `.reg` files never connect anywhere; they only edit local registry
keys (some of which change how Windows Update itself behaves).
