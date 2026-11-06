# 从零部署 Token Monitor + System vitals

这份指南面向**还没有安装 Token Monitor** 的用户，覆盖 macOS、Windows 和 Linux 的差异、安装 / 升级命令、插件信任流程、验证和排错。已经装好 Token Monitor ≥ 0.82.0 时，可以直接从[第 2 步](#第-2-步安装-system-vitals)开始。

> 名称对照：Token Monitor 中文界面把 Edge Dock 叫作「侧边栏」，插件在中文界面显示为「系统负载」。下文统一写 Edge Dock，并在首次出现时附中文名。

## 适用范围

| 组合 | 是否支持 | 说明 |
| --- | --- | --- |
| macOS 13+（Apple Silicon） | 支持，已实测 | 插件在 0.85.0 / macOS arm64 上实测 |
| macOS 13+（Intel） | 理论上支持，未实测 | 插件没有原生依赖，但未在 Intel Mac 上验证 |
| Windows 10/11 x64 | 支持，未实测 | Edge Dock 在 Windows 可用；内存使用 `os.freemem()` 口径 |
| Linux x64 | **不支持本插件** | Token Monitor 本体可运行，但 Edge Dock 仅在 macOS / Windows 提供，而本插件只声明了 Edge Dock 视图 |

最低版本：Token Monitor `0.82.0`（插件 manifest 的 `engines.tokenMonitor`）。建议直接安装最新稳定版；App 会自动检查更新。

插件本身不访问网络。安装 App 和下载 `.tmplugin` 需要能访问 GitHub Releases；离线 / 内网步骤见[离线安装](#离线--内网安装)。

## 准备

- macOS 13 或更高版本；Windows 10/11 x64；Linux 不支持本插件。
- 安装 App 需要管理员权限或可写的 `/Applications`（macOS）；Windows 安装程序默认按当前用户安装。
- 能访问 `github.com` 和 `objects.githubusercontent.com`（Release 资产会重定向到后者）。
- 可选：`gh` CLI 或 `curl`，用于命令行下载和校验；没有也能在浏览器里手动完成。
- 如果你要启用插件，请确认你信任这个包：插件不是沙盒，Node 后端以你的操作系统用户权限运行。

## 第 1 步：安装 Token Monitor

### macOS：Homebrew（推荐）

```bash
brew install --cask token-monitor
```

Homebrew Cask 可能比 GitHub Releases 晚一两个版本，只要不低于 `0.82.0` 就可以安装本插件。App 装好后会自行提示更新；如果 Cask 版本低于 `0.82.0`，改用下面的 DMG 安装最新版。

升级：

```bash
brew upgrade --cask token-monitor
```

### macOS：直接下载 DMG

从 [Token Monitor Releases](https://github.com/Javis603/token-monitor/releases/latest) 下载与架构对应的文件：

| 架构 | 资产 |
| --- | --- |
| Apple Silicon | `Token-Monitor-<version>-arm64.dmg` |
| Intel | `Token-Monitor-<version>-x64.dmg` |

手动安装：双击 DMG，把 **Token Monitor.app** 拖进 **Applications**。

命令行 / 自动化安装：

```bash
set -euo pipefail

tmp="$(mktemp -d)"
mkdir -p "$tmp/mnt"

case "$(uname -m)" in
  arm64)  pattern='Token-Monitor-*-arm64.dmg' ;;
  x86_64) pattern='Token-Monitor-*-x64.dmg' ;;
  *) echo "Unsupported architecture: $(uname -m)" >&2; exit 1 ;;
esac

gh release download --repo Javis603/token-monitor --pattern "$pattern" --dir "$tmp" --clobber
dmg="$(find "$tmp" -maxdepth 1 -name '*.dmg' -print -quit)"

# 先退出正在运行的 App；没安装时会失败，忽略即可
osascript -e 'tell application "Token Monitor" to quit' >/dev/null 2>&1 || true

hdiutil attach "$dmg" -nobrowse -quiet -mountpoint "$tmp/mnt"
ditto "$tmp/mnt/Token Monitor.app" "/Applications/Token Monitor.app"
hdiutil detach "$tmp/mnt"

open -a "Token Monitor"
```

如果 `/Applications` 不可写，把目标改成 `"$HOME/Applications/Token Monitor.app"`，或使用 `sudo ditto`。使用 `ditto` 而不是 `cp -R`，升级时才会覆盖旧 App，而不是嵌套一份新副本。

没有 `gh` 时，在 Releases 页手动下载 DMG，然后从 `hdiutil attach` 开始执行。

### Windows 10/11

从 [Releases](https://github.com/Javis603/token-monitor/releases/latest) 下载：

- **安装版**：`Token-Monitor-Setup-<version>.exe`（NSIS，默认按当前用户安装，可选安装目录）
- **便携版**：`Token-Monitor-<version>.exe`（不需要安装，但每次从同一位置运行才能复用同一份用户数据）

PowerShell 下载并启动安装程序：

```powershell
$root = Join-Path $env:TEMP 'token-monitor-install'
New-Item -ItemType Directory -Force -Path $root | Out-Null

gh release download Javis603/token-monitor --pattern 'Token-Monitor-Setup-*.exe' --dir $root --clobber
$setup = Get-ChildItem $root -Filter 'Token-Monitor-Setup-*.exe' |
  Sort-Object LastWriteTime -Descending | Select-Object -First 1

Start-Process -Wait -FilePath $setup.FullName
```

没有 `gh` 时：

```powershell
$release = Invoke-RestMethod 'https://api.github.com/repos/Javis603/token-monitor/releases/latest'
$asset = $release.assets |
  Where-Object { $_.name -like 'Token-Monitor-Setup-*.exe' } |
  Select-Object -First 1
Invoke-WebRequest -Uri $asset.browser_download_url -OutFile (Join-Path $root $asset.name)
Start-Process -Wait -FilePath (Join-Path $root $asset.name)
```

NSIS 安装器支持 `/S` 静默安装，但安装目录和快捷方式行为取决于安装器选项；CI / 自动化场景建议安装后用下面的注册表查询确认位置，不要假设固定路径。

### Linux

Token Monitor 本体提供 x64 AppImage：

```bash
chmod +x Token-Monitor-<version>.AppImage
./Token-Monitor-<version>.AppImage
```

**但本插件在 Linux 上不可用**：它只声明了 `edgeDock` 视图，而 Edge Dock 仅 macOS / Windows 提供。不要继续安装 `.tmplugin`；Token Monitor 本体仍可照常使用。

### 确认版本与首次启动

macOS：

```bash
/usr/libexec/PlistBuddy -c 'Print :CFBundleShortVersionString' "/Applications/Token Monitor.app/Contents/Info.plist"
```

Windows PowerShell：

```powershell
$entry = Get-ItemProperty 'HKCU:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*' -ErrorAction SilentlyContinue |
  Where-Object { $_.DisplayName -like 'Token Monitor*' } |
  Select-Object -First 1

if ($entry) {
  $entry.DisplayVersion
  $entry.InstallLocation
} else {
  'Not found in the per-user uninstall registry'
}
```

版本必须 ≥ `0.82.0`。首次启动默认是**本地模式**，不需要 hub、agent 或配置文件；App 会开始追踪本机用量。

## 第 2 步：安装 System vitals

### 1. 下载

当前版本：`system-vitals-1.0.1.tmplugin`。

- 最新 Release：<https://github.com/swingmonkey/token-monitor-plugin-system-vitals/releases/latest>
- 固定版本直链：<https://github.com/swingmonkey/token-monitor-plugin-system-vitals/releases/download/v1.0.1/system-vitals-1.0.1.tmplugin>
- 校验文件：<https://github.com/swingmonkey/token-monitor-plugin-system-vitals/releases/download/v1.0.1/SHA256SUMS>

命令行（macOS / Linux，使用 `gh`）：

```bash
tmp="${TMPDIR:-/tmp}/system-vitals"
mkdir -p "$tmp"
gh release download --repo swingmonkey/token-monitor-plugin-system-vitals \
  --pattern 'system-vitals-*.tmplugin' --dir "$tmp" --clobber
ls -l "$tmp"
```

没有 `gh` 时：

```bash
curl -fsSL https://api.github.com/repos/swingmonkey/token-monitor-plugin-system-vitals/releases/latest \
  | python3 -c 'import json,sys; [print(a["name"], a.get("digest",""), a["browser_download_url"]) for a in json.load(sys.stdin)["assets"] if a["name"].endswith(".tmplugin")]'
```

然后用 `curl -fL <browser_download_url> -o system-vitals.tmplugin` 下载。

Windows PowerShell：

```powershell
$root = Join-Path $env:TEMP 'system-vitals'
New-Item -ItemType Directory -Force -Path $root | Out-Null

$release = Invoke-RestMethod 'https://api.github.com/repos/swingmonkey/token-monitor-plugin-system-vitals/releases/latest'
$asset = $release.assets | Where-Object { $_.name -like '*.tmplugin' } | Select-Object -First 1
Invoke-WebRequest -Uri $asset.browser_download_url -OutFile (Join-Path $root $asset.name)
$asset.name
$asset.digest
```

### 2. 校验

把本地文件的 SHA-256 与 Release 资产的 digest 对比：

```bash
# macOS / Linux
shasum -a 256 "$tmp"/*.tmplugin
```

```powershell
# Windows PowerShell
Get-FileHash (Join-Path $root $asset.name) -Algorithm SHA256
```

当前版本的 SHA-256 见 Release 资产的 digest，或下载同一 Release 中的 `SHA256SUMS` 对比。GitHub 的 digest 通常写成 `sha256:<hex>`；`shasum` / `Get-FileHash` 只输出 `<hex>`。比较时忽略 `sha256:` 前缀和大小写。

`gh` 可以直接列出资产 digest：

```bash
gh release view --repo swingmonkey/token-monitor-plugin-system-vitals \
  --json assets --jq '.assets[] | select(.name|endswith(".tmplugin")) | "\(.name) \(.digest)"'
```

哈希不一致时不要安装，重新下载。GitHub Releases 页面显示的 SHA-256 / digest 是判断依据；不要把完整性哈希当成安全审计。

### 3. 导入并信任

1. 打开 Token Monitor 主窗口，点右下角 ⚙ 打开设置。
2. 进入 **Plugins（插件）**。
3. 确认 **Enable plugins（启用插件）** 已打开（在 **More plugin actions（更多插件操作）** 菜单中）。若顶部显示 **Safe mode（安全模式）**：
   - 完全退出 Token Monitor，再正常重新打开；
   - 如果 App 是用 `--plugins-safe-mode` 启动的，去掉该参数；
   - 如果设置了环境变量 `TOKEN_MONITOR_PLUGINS_SAFE_MODE=1`，先移除；
   - 不要使用菜单里的 **Restart without plugins（不加载插件并重新启动）**，那会再次进入安全模式。
4. 切到 **Installed（已安装）**，点 **Install from file（从文件安装）**。
5. 选择 `.tmplugin` 文件。App 会显示包名、版本、作者、类型 / 后台运行说明和 **Package SHA-256（软件包 SHA-256）**。
6. 选择：
   - **Trust, install and enable（信任并安装启用）**：立即启用并运行；
   - **Install only（只安装）**：只复制文件，保持停用，稍后打开开关时再信任并进入首次设置。

   App 需要用户明确确认信任。没有命令行参数可以跳过这一步。

### 4. 首次设置

启用时会显示 **First-use settings（首次设置）**。建议：

- 保留 **CPU ring / Memory ring（CPU 圆环 / 内存圆环）** 的 Edge Dock 放置勾选；
- 希望 Edge Dock 收起后仍持续采样时，保留 **Run in background（后台执行）**。

这些默认值来自插件的 `defaultSurfaces` 和 `defaults.background`。确认后，两个视图会自动加入 Edge Dock。以后重新启用或更新时，App 会保留你的选择。

### 5. 打开 Edge Dock

如果 Edge Dock 没有显示：

- **Settings（设置）→ Window（窗口）→ Floating & Tray（浮窗与系统托盘）→ Edge Dock（侧边栏）** 勾选；
- 或从菜单栏 / 系统托盘的 **Edge Dock（侧边栏）→ Show Edge Dock（显示侧边栏）** 打开。

默认是自动隐藏：把鼠标推到屏幕边缘唤出，点击可固定。放置不影响 Edge Dock 的总开关，必须单独打开。

## 验证

### 界面

打开 **Plugins → Installed**：

- 插件显示为 **System vitals（系统负载）**；
- 状态为 **On**；
- 没有 **Problems（问题）**；
- 运行状态为 **Running**；如果关闭了后台执行且当前没有可见视图，**Idle** 正常。

然后确认：

- Edge Dock 中有 **CPU** 和 **内存 / MEM** 两个圆环；
- 数值约每 2 秒刷新；
- 悬停圆环出现卡片：大数字、近期曲线、CPU 卡的系统负载均值或内存 已用 / 总量；
- 占用 ≥ 70% 显示橙色，≥ 90% 显示红色。

### 文件系统（可选，适合 agent / 排错）

安装成功后，Token Monitor 会在用户数据目录下保留托管插件和注册表：

| 平台 | 用户数据目录 |
| --- | --- |
| macOS | `~/Library/Application Support/Token Monitor` |
| Windows | `%APPDATA%\Token Monitor` |
| Linux | `~/.config/Token Monitor`（本插件不适用） |

- 插件文件：`plugins/system-vitals/`
- 托管注册表：`plugin-manager/installed.json`
- 应用设置：`settings.json`（**不要手动编辑**）

macOS 示例：

```bash
root="$HOME/Library/Application Support/Token Monitor"
ls "$root/plugins/system-vitals"
python3 -c 'import json,os; d=json.load(open(os.path.expanduser("~/Library/Application Support/Token Monitor/plugin-manager/installed.json"))); print(d.get("system-vitals"))'
```

Windows PowerShell：

```powershell
$root = Join-Path $env:APPDATA 'Token Monitor'
Test-Path (Join-Path $root 'plugins\system-vitals')
Get-Content (Join-Path $root 'plugin-manager\installed.json') -Raw | ConvertFrom-Json |
  Select-Object -ExpandProperty 'system-vitals'
```

注册表条目应包含类似字段：

```json
{
  "version": "1.0.1",
  "sha256": "<release-sha256>",
  "origin": { "id": "local", "kind": "local" }
}
```

这只能证明包已经安装。是否启用、是否真正放置到 Edge Dock，以 App 界面为准。

## 升级、回滚、卸载

### 升级 App

- macOS Homebrew：`brew upgrade --cask token-monitor`
- 其他平台：App 内的更新提示，或从 Releases 下载新版本覆盖安装。
- App 升级不会自动升级插件，也不会重置插件的信任状态。

### 升级插件

1. 下载新版本 `system-vitals-<version>.tmplugin`；
2. 按第 2 步导入；
3. App 会显示新版本的包信息和 SHA-256，要求重新确认信任（**Trust and update / 信任并更新**）；
4. 已启用的插件在更新后恢复启用，已关闭的保持关闭；放置位置和设置保留。

### 回滚插件

在插件卡片上点 **Settings（设置）→ Roll back（回滚）**，恢复上一版代码。回滚不会撤销插件在 App 外造成的副作用；本插件没有持久化数据，历史曲线只存在内存中。

### 停用 / 卸载

- **停用**：关闭插件开关，后端停止采样。
- **卸载**：插件卡片 → **Settings（设置）→ Uninstall（卸载）**。可以选择保留插件数据和放置位置，方便以后重装。
- **界面卡死**：菜单栏 / 系统托盘 → **Restart without plugins（不加载插件并重新启动）**，或用 `--plugins-safe-mode` / `TOKEN_MONITOR_PLUGINS_SAFE_MODE=1` 启动。安全模式只管理包，不加载插件代码。

## 排错

| 现象 | 检查 | 处理 |
| --- | --- | --- |
| 安装按钮不可用 | 插件总开关、安全模式 | 打开 **Enable plugins**；退出安全模式后重启 App |
| `incompatible-plugin-api` / `incompatible-plugin-package` | App 版本、包格式 | 升级到 ≥ 0.82.0；确认下载的是 `.tmplugin`，不是源码 zip |
| 导入时提示包损坏 / 哈希不符 | 下载完整性 | 重新下载并校验 SHA-256 |
| 安装后没有圆环 | Edge Dock 总开关、放置位置 | 打开 Edge Dock；在 Plugins → System vitals → Show on 勾选 |
| 首次设置只放置了一个圆环 | Edge Dock 已满（24 项） | 移除一项后重新勾选 |
| 插件显示 Off / Needs approval | 只安装未启用；更新后未重新信任 | 打开开关；点 **Review updated code（检查更新后的代码）** |
| 圆环一直显示 `—` | 后端未启动 / 第一次采样未完成 | 等 2–5 秒；插件卡片 → **Settings（设置）→ Restart（重启）** |
| 运行状态 Idle | 后台执行关闭且没有可见视图 | 正常；打开 Edge Dock 或开启后台执行 |
| Windows 内存百分比与任务管理器不同 | `os.freemem()` 口径 | 预期行为；只有 macOS 使用 `vm_stat` |
| macOS 内存短时间不更新 | `vm_stat` 读取失败，沿用上次成功读数 | 几秒内恢复；持续异常时重启插件 |
| 圆环颜色不变 | 未达到 70% / 90% | 正常 |
| Linux 找不到 Edge Dock | 平台不支持 | 本插件仅 macOS / Windows |
| 卸载后仍有圆环 | 只卸载了文件，放置未移除 | 卸载时选择不保留放置位置，或手动移除 Edge Dock 项目 |

如果 App 无法启动或插件界面导致 App 卡死，优先使用安全模式；不要删除 `plugin-manager` 或手改 `settings.json`，那会绕过事务恢复和信任检查。

## 离线 / 内网安装

1. 在一台能上网的机器上下载：
   - Token Monitor 安装包（按目标平台 / 架构）；
   - `system-vitals-<version>.tmplugin`；
   - 记录两个文件的 SHA-256。
2. 通过内网文件服务器 / U 盘传输，并在目标机器上重新校验哈希。
3. 安装 App（DMG / EXE / AppImage），启动一次。
4. 在 App 内用 **Install from file** 导入 `.tmplugin`。
5. 插件不需要网络；App 的用量采集和插件采样都在本机完成。

不能只复制 `plugins/system-vitals` 到目标机器来「安装」：托管安装需要 `plugin-manager/installed.json` 和用户的信任确认，手工拼接会绕过安全检查，且更新 / 回滚不可用。

## 版本与资产速查

| 项目 | 值 |
| --- | --- |
| 插件 manifest id | `system-vitals` |
| 插件 API | `tmApi: 1` |
| 包格式 | `format: 2` |
| 最低 App | `0.82.0` |
| 插件资产 | `system-vitals-<version>.tmplugin` |
| App 资产（macOS） | `Token-Monitor-<version>-arm64.dmg` / `-x64.dmg` |
| App 资产（Windows） | `Token-Monitor-Setup-<version>.exe` / `Token-Monitor-<version>.exe` |
| App 资产（Linux） | `Token-Monitor-<version>.AppImage`（本插件不可用） |

## 给 AI agent

如果部署由 coding agent 执行，让它阅读 [AGENTS.md](../AGENTS.md)。其中包含：

- 平台 / 架构 / 版本预检命令；
- macOS 和 Windows 的下载与安装命令；
- 哈希校验；
- 必须交给用户完成的信任步骤；
- 安装后的界面和文件系统验证；
- 失败时的停止条件和恢复方式。

再强调一次：**插件信任必须在 Token Monitor 界面中由用户确认**。不要修改 `settings.json`、`plugin-manager/installed.json` 或插件的 `trustedRevision` 来绕过信任，也不要把文件直接复制到 `plugins/` 目录冒充安装。
