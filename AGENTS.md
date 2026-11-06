# AGENTS.md — deploy Token Monitor + System vitals

This repository ships **System vitals**, a plugin for [Token Monitor](https://github.com/Javis603/token-monitor). It does not run standalone. This file is a deployment runbook for coding agents that are asked to install or upgrade both the app and the plugin.

Read the user-facing guide for context: [docs/install.md](docs/install.md).

## Goal

On macOS 13+ or Windows 10/11 x64:

1. Token Monitor is installed and is at least `0.82.0`.
2. `system-vitals-<version>.tmplugin` is downloaded and its SHA-256 matches the GitHub Release asset digest.
3. The user has reviewed and trusted the package in the Token Monitor UI.
4. The plugin is enabled and both rings (CPU / Memory) are placed in Edge Dock.

Linux is out of scope: the plugin declares only `edgeDock` views, and Edge Dock is unavailable on Linux. Stop and explain instead of installing the plugin.

## Hard rules

- **The trust decision belongs to the user.** Token Monitor has no CLI for importing or trusting plugins. Do not modify `settings.json`, `plugin-manager/installed.json`, `trustedRevision`, or copy files into `plugins/` to bypass the UI. Those changes bypass a security boundary and break updates/rollback.
- **Do not claim success without evidence.** A downloaded file is not an installed plugin. Require the UI to show the plugin `On`, or a user confirmation plus the file-system checks below.
- **Verify the hash before asking for trust.** Compare the local `.tmplugin` SHA-256 with the release asset `digest`.
- **Confirm before disruptive actions** if the user has not already authorized installation: installing system software, quitting a running app, or replacing `/Applications/Token Monitor.app`.
- **Do not install the plugin on Linux.** Do not attempt to add an Edge Dock surface that the plugin does not declare.
- **Do not invent commands.** If `gh`, Homebrew, or the expected install path is missing, use the documented fallback or hand off.

## Preflight

Collect and report:

- OS and version
- CPU architecture
- Token Monitor installed? Path and version
- Plugin already installed? Version
- Available tools: `gh`, `brew` (macOS), `curl`, `python3`
- Whether the user is able to click through the Token Monitor UI

### macOS

```bash
uname -s
uname -m
sw_vers -productVersion
command -v brew || echo "brew: missing"
command -v gh || echo "gh: missing"

APP="/Applications/Token Monitor.app"
if [ -d "$APP" ]; then
  /usr/libexec/PlistBuddy -c 'Print :CFBundleShortVersionString' "$APP/Contents/Info.plist"
else
  echo "Token Monitor: missing"
fi
```

Requirements: `Darwin`, macOS `13.0+`, `arm64` or `x86_64`. The plugin is tested on macOS arm64 with Token Monitor `0.85.0`; newer `0.8x` releases should satisfy the manifest's `>=0.82.0` engine range.

### Windows PowerShell

```powershell
$PSVersionTable.PSVersion
[Environment]::Is64BitOperatingSystem
(Get-CimInstance Win32_OperatingSystem).Caption
Get-Command gh -ErrorAction SilentlyContinue | Select-Object -ExpandProperty Source

$entry = Get-ItemProperty 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*' -ErrorAction SilentlyContinue |
  Where-Object { $_.DisplayName -like 'Token Monitor*' } |
  Select-Object -First 1
if ($entry) { $entry.DisplayVersion; $entry.InstallLocation } else { 'Token Monitor: missing' }
```

Requirements: Windows 10/11 x64. The plugin is not tested on Windows; say so in the final report.

## Step 1 — install or upgrade Token Monitor

### macOS, Homebrew path (preferred)

```bash
if [ ! -d "/Applications/Token Monitor.app" ]; then
  brew install --cask token-monitor
else
  brew upgrade --cask token-monitor
fi
```

Homebrew Cask may lag GitHub Releases by one or two versions. Any version `>=0.82.0` is acceptable. The app updates itself.

### macOS, direct DMG fallback

Use this when Homebrew is unavailable or the Cask is too old.

```bash
set -euo pipefail

tmp="$(mktemp -d)"
mkdir -p "$tmp/mnt"

case "$(uname -m)" in
  arm64)  pattern='Token-Monitor-*-arm64.dmg' ;;
  x86_64) pattern='Token-Monitor-*-x64.dmg' ;;
  *) echo "Unsupported architecture: $(uname -m)" >&2; exit 1 ;;
esac

if command -v gh >/dev/null 2>&1; then
  gh release download --repo Javis603/token-monitor --pattern "$pattern" --dir "$tmp" --clobber
else
  echo "gh is missing; download the matching DMG from https://github.com/Javis603/token-monitor/releases/latest into $tmp" >&2
  exit 2
fi

dmg="$(find "$tmp" -maxdepth 1 -name '*.dmg' -print -quit)"

osascript -e 'tell application "Token Monitor" to quit' >/dev/null 2>&1 || true

hdiutil attach "$dmg" -nobrowse -quiet -mountpoint "$tmp/mnt"
ditto "$tmp/mnt/Token Monitor.app" "/Applications/Token Monitor.app"
hdiutil detach "$tmp/mnt"

open -a "Token Monitor"
```

If `/Applications` is not writable, use `sudo ditto` (after user confirmation) or install to `$HOME/Applications`. Do not use `cp -R` for upgrades; `ditto` replaces the bundle instead of nesting a copy.

### Windows

```powershell
$root = Join-Path $env:TEMP 'token-monitor-install'
New-Item -ItemType Directory -Force -Path $root | Out-Null

if (Get-Command gh -ErrorAction SilentlyContinue) {
  gh release download Javis603/token-monitor --pattern 'Token-Monitor-Setup-*.exe' --dir $root --clobber
  $setup = Get-ChildItem $root -Filter 'Token-Monitor-Setup-*.exe' |
    Sort-Object LastWriteTime -Descending | Select-Object -First 1
} else {
  $release = Invoke-RestMethod 'https://api.github.com/repos/Javis603/token-monitor/releases/latest'
  $asset = $release.assets |
    Where-Object { $_.name -like 'Token-Monitor-Setup-*.exe' } |
    Select-Object -First 1
  $target = Join-Path $root $asset.name
  Invoke-WebRequest -Uri $asset.browser_download_url -OutFile $target
  $setup = Get-Item $target
}

Start-Process -Wait -FilePath $setup.FullName
```

The installer is interactive by default. `/S` is NSIS's silent flag, but the install directory is user-configurable; do not assume a fixed path. A portable build (`Token-Monitor-<version>.exe`, no `Setup` in the name) may be used if the user prefers no installation.

After install, locate the executable and version:

```powershell
$entry = Get-ItemProperty 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*' -ErrorAction SilentlyContinue |
  Where-Object { $_.DisplayName -like 'Token Monitor*' } |
  Select-Object -First 1

if ($entry -and $entry.InstallLocation) {
  $exe = Join-Path $entry.InstallLocation 'Token Monitor.exe'
} else {
  $exe = (Get-ChildItem "$env:LOCALAPPDATA\Programs","$env:ProgramFiles" -Filter 'Token Monitor.exe' -Recurse -ErrorAction SilentlyContinue |
    Select-Object -First 1).FullName
}

if (-not $exe) { throw 'Token Monitor.exe not found; ask the user for the install path' }
$entry.DisplayVersion
$exe
```

Require `>=0.82.0`. Launch:

```powershell
Start-Process $exe
```

### Linux

Do not proceed with the plugin. Token Monitor can run from an AppImage, but Edge Dock is unavailable on Linux and the plugin has no other surface. Report this limitation.

## Step 2 — download and verify the plugin

### macOS / Linux shell

```bash
tmp="${TMPDIR:-/tmp}/system-vitals"
mkdir -p "$tmp"

if command -v gh >/dev/null 2>&1; then
  gh release download --repo swingmonkey/token-monitor-plugin-system-vitals \
    --pattern 'system-vitals-*.tmplugin' --dir "$tmp" --clobber
else
  echo "gh is missing; use the GitHub API / release page to download the .tmplugin into $tmp" >&2
  exit 2
fi

pkg="$(find "$tmp" -maxdepth 1 -name '*.tmplugin' -print -quit)"
shasum -a 256 "$pkg"
```

Expected digest from the release:

```bash
gh release view --repo swingmonkey/token-monitor-plugin-system-vitals \
  --json assets --jq '.assets[] | select(.name|endswith(".tmplugin")) | "\(.name) \(.digest)"'
```

Compare the local hash with the release asset `digest`, or download the `SHA256SUMS` asset from the same release. `v1.0.2` is the current release.

If `gh` is unavailable:

```bash
curl -fsSL https://api.github.com/repos/swingmonkey/token-monitor-plugin-system-vitals/releases/latest \
  | python3 -c 'import json,sys; [print(a["name"], a.get("digest",""), a["browser_download_url"]) for a in json.load(sys.stdin)["assets"] if a["name"].endswith(".tmplugin")]'
```

### Windows PowerShell

```powershell
$root = Join-Path $env:TEMP 'system-vitals'
New-Item -ItemType Directory -Force -Path $root | Out-Null

if (Get-Command gh -ErrorAction SilentlyContinue) {
  gh release download swingmonkey/token-monitor-plugin-system-vitals --pattern 'system-vitals-*.tmplugin' --dir $root --clobber
  $pkg = Get-ChildItem $root -Filter 'system-vitals-*.tmplugin' |
    Sort-Object LastWriteTime -Descending | Select-Object -First 1
} else {
  $release = Invoke-RestMethod 'https://api.github.com/repos/swingmonkey/token-monitor-plugin-system-vitals/releases/latest'
  $asset = $release.assets | Where-Object { $_.name -like '*.tmplugin' } | Select-Object -First 1
  $target = Join-Path $root $asset.name
  Invoke-WebRequest -Uri $asset.browser_download_url -OutFile $target
  $asset.digest
  $pkg = Get-Item $target
}

Get-FileHash $pkg.FullName -Algorithm SHA256
```

Compare the file hash to the release digest. If they differ, re-download and stop if it still differs.

## Step 3 — required human handoff

The app has **no CLI** for plugin import or trust. After the app is installed and the `.tmplugin` is verified:

1. Reveal the package and open the app:
   - macOS: `open -R "$pkg"` then `open -a "Token Monitor"`
   - Windows: `Start-Process explorer.exe "/select,$($pkg.FullName)"` then `Start-Process $exe`
2. Show the user this exact path:
   - Open the Token Monitor main window → bottom-right ⚙ → **Plugins**
   - Confirm **Enable plugins** (in the **More plugin actions** menu) is on
   - If **Safe mode** is shown: fully quit the app and reopen normally; remove `--plugins-safe-mode` or `TOKEN_MONITOR_PLUGINS_SAFE_MODE=1` if present
   - **Installed** → **Install from file** → select the `.tmplugin`
   - Review the package and SHA-256 → **Trust, install and enable**
   - In **First-use settings**, keep both ring placements in **Edge Dock** and keep **Run in background** if the user wants continuous sampling
   - If Edge Dock is off: **Settings → Window → Floating & Tray → Edge Dock** (Chinese UI: 设置 → 窗口 → 浮窗与系统托盘 → 侧边栏)
3. Wait for the user to confirm. Do not proceed to a success claim before that.

If the user prefers **Install only**, the plugin remains disabled; report that state accurately and do not enable it yourself.

## Step 4 — verify

### UI (authoritative)

Ask the user to confirm, or inspect via an authorized screenshot/GUI tool:

- **Plugins → Installed** shows **System vitals / 系统负载**, status **On**, no **Problems**
- Runtime is **Running**; **Idle** is acceptable when background is off and no view is visible
- Edge Dock shows **CPU** and **内存 / MEM** rings
- Hovering shows the card; values refresh about every 2 seconds

### File system (corroboration only)

| Platform | User data directory |
| --- | --- |
| macOS | `~/Library/Application Support/Token Monitor` |
| Windows | `%APPDATA%\Token Monitor` |

Expected:

- `plugins/system-vitals/` exists
- `plugin-manager/installed.json` contains a `system-vitals` entry with the expected `version` and `sha256`
- Do not parse `settings.json` as a public contract; do not edit it

macOS:

```bash
root="$HOME/Library/Application Support/Token Monitor"
test -d "$root/plugins/system-vitals"
python3 -c 'import json,os; d=json.load(open(os.path.expanduser("~/Library/Application Support/Token Monitor/plugin-manager/installed.json"))); print(d.get("system-vitals"))'
```

Windows:

```powershell
$root = Join-Path $env:APPDATA 'Token Monitor'
Test-Path (Join-Path $root 'plugins\system-vitals')
Get-Content (Join-Path $root 'plugin-manager\installed.json') -Raw | ConvertFrom-Json |
  Select-Object -ExpandProperty 'system-vitals'
```

The registry proves the package is installed, not that the user trusted it or placed the views.

## Failure handling

| Failure | Action |
| --- | --- |
| Linux host | Stop; the plugin cannot display there |
| App version < 0.82.0 | Upgrade via Homebrew, in-app update, or the latest release; re-check |
| `gh` missing | Use the documented PowerShell / curl API fallback, or hand off the release URL |
| Homebrew missing / no admin rights | Use the direct DMG fallback or ask the user to install manually |
| SHA-256 mismatch | Re-download once; if it still differs, stop and report the expected vs actual hash |
| Install button disabled | Plugins globally off or safe mode; guide the user, do not edit settings |
| Package incompatible | App too old; upgrade |
| No rings after install | Edge Dock master switch off, placement skipped (dock full), or plugin off; check each in the UI |
| Edge Dock full (24 items) | Ask the user to remove an item, then place the rings via **Show on** |
| User declines trust | Stop; report "not trusted / not enabled" |
| GUI cannot be driven | Hand off; state that the remaining steps are manual and that success is unverified |
| App crashes or UI freezes | Use **Restart without plugins** / safe mode; do not delete plugin-manager data |

## Report format

```text
Status: installed | needs human action | blocked
Platform: macOS <version> <arch> | Windows <version> x64
App: <version> at <path>
Plugin file: <name>, SHA-256 <hash> (matches release: yes/no)
Registry: version <version>, installed: yes/no
Trust/enable: user-confirmed yes/no (never infer)
Placement: CPU ring in Edge Dock yes/no; Memory ring yes/no
Tests: UI observed / user-confirmed / unverified
Next step: <one concrete action or "none">
```

Include the exact UI path in the handoff so the user does not have to search for it. If the user is on Linux, say the plugin is unsupported rather than offering a partial install.
