# OpenClaw 在 Windows 上从 0 到可运行全流程（本机实操版）

本文记录本机在 Windows PowerShell 下，把 openclaw/openclaw 源码跑起来的完整流程。

## 0. 真实运行时配置路径总览（你要找的就是这里）

OpenClaw 在 Windows 上有两层目录：

1. `C:\Users\aaape\.openclaw-dev`：网关运行时配置与运行数据
2. `C:\Users\aaape\.openclaw\workspace-dev`：Agent 人格/画像/规则文件（SOUL/USER/TOOLS/AGENTS）

### 0.1 运行时配置主文件

- 真正的运行时配置文件：`C:\Users\aaape\.openclaw-dev\openclaw.json`
- 最近一次可回滚备份：`C:\Users\aaape\.openclaw-dev\openclaw.json.last-good`

### 0.2 运行时目录（.openclaw-dev）

- `C:\Users\aaape\.openclaw-dev\openclaw.json`：网关配置入口
- `C:\Users\aaape\.openclaw-dev\logs`：运行日志
- `C:\Users\aaape\.openclaw-dev\tasks`：任务相关运行数据
- `C:\Users\aaape\.openclaw-dev\identity`：身份相关运行数据
- `C:\Users\aaape\.openclaw-dev\acpx`：协议/扩展运行数据
- `C:\Users\aaape\.openclaw-dev\gateway-instance-id`：当前网关实例标识

### 0.3 Persona/画像目录（workspace-dev）

- `C:\Users\aaape\.openclaw\workspace-dev\SOUL.md`：Agent 核心人格与行为基调
- `C:\Users\aaape\.openclaw\workspace-dev\USER.md`：用户偏好与画像信息
- `C:\Users\aaape\.openclaw\workspace-dev\IDENTITY.md`：身份设定
- `C:\Users\aaape\.openclaw\workspace-dev\AGENTS.md`：agent 组织与约束
- `C:\Users\aaape\.openclaw\workspace-dev\TOOLS.md`：工具策略/使用规则
- `C:\Users\aaape\.openclaw\workspace-dev\state`：运行状态数据

### 0.4 一键查看命令（复制即用）

```powershell
$userHome=[Environment]::GetFolderPath('UserProfile')

# 查看运行时配置文件
Get-Content "$userHome\.openclaw-dev\openclaw.json" -TotalCount 120

# 查看 .openclaw-dev 顶层
Get-ChildItem -Force "$userHome\.openclaw-dev"

# 查看 persona/workspace 文件
Get-ChildItem -Force "$userHome\.openclaw\workspace-dev"
```

### 0.5 你工作区里的快照文件

为了便于在当前项目里直接查看，我已复制一份运行时配置快照到：

- `d:\aihackton\openclaw.runtime.snapshot.json`

说明：

- 这个快照用于查看与备份。
- 真正生效的仍然是：`C:\Users\aaape\.openclaw-dev\openclaw.json`。

## 1. 目标与现状

- 目标：在本机源码目录启动 OpenClaw Gateway。
- 仓库目录：`d:\aihackton\openclaw-main`
- 启动后可用验证：
  - `http://127.0.0.1:19001/health` 返回 `{"ok":true,"status":"live"}`

## 2. 关键背景（为什么要这样做）

本机初始状态：

- 默认 Node 是 `v18.20.0`（不满足 OpenClaw 要求）。
- 项目要求 Node `>=22.16`（推荐 Node 24）。
- PowerShell 执行策略会拦截 `pnpm.ps1`。
- `package.json` 的 `gateway:dev` 脚本使用了类 Unix 环境变量写法（在 PowerShell 直接跑会失败）。

因此采用方案：

1. 不改系统默认 Node，额外放一个便携 Node 24 到 `d:\environment\nodejs24`。
2. 使用 `pnpm.cmd`（避免 `pnpm.ps1` 执行策略问题）。
3. 用 PowerShell 原生方式设置环境变量再启动 Gateway。

## 3. 一次性准备：Node 24 便携运行时

在 PowerShell 执行：

```powershell
$base='d:\environment'
$zip=Join-Path $base 'node-v24.15.0-win-x64.zip'
$extractRoot=Join-Path $base 'node24_tmp'
$target=Join-Path $base 'nodejs24'

if (Test-Path $zip) { Remove-Item -Force $zip }
if (Test-Path $extractRoot) { Remove-Item -Recurse -Force $extractRoot }
if (Test-Path $target) { Remove-Item -Recurse -Force $target }

curl.exe -L "https://nodejs.org/dist/v24.15.0/node-v24.15.0-win-x64.zip" -o $zip
Expand-Archive -Path $zip -DestinationPath $extractRoot -Force
$inner=(Get-ChildItem $extractRoot -Directory | Select-Object -First 1).FullName
Move-Item -Path $inner -Destination $target
Remove-Item -Recurse -Force $extractRoot

& 'd:\environment\nodejs24\node.exe' -v
& 'd:\environment\nodejs24\npm.cmd' -v
```

预期版本：

- Node：`v24.15.0`
- npm：`11.x`

## 4. 准备 pnpm（PowerShell 兼容方式）

```powershell
$env:Path='d:\environment\nodejs24;' + $env:Path
corepack enable
corepack prepare pnpm@10.18.3 --activate
pnpm.cmd -v
```

说明：

- 如果你看到询问是否下载 pnpm，输入 `Y`。
- 即使 `pnpm.ps1` 被执行策略拦截，`pnpm.cmd` 仍可正常使用。

## 5. 安装项目依赖

```powershell
Set-Location d:\aihackton\openclaw-main
$env:Path='d:\environment\nodejs24;' + $env:Path
pnpm.cmd install
```

预期：

- 输出包含 `Scope: all ... workspace projects`
- 最后出现 `Done in ... using pnpm ...`

## 6. 启动 Gateway（Windows 正确写法）

不要直接用 `pnpm gateway:dev`（会因为环境变量写法失败）。

请用：

```powershell
Set-Location d:\aihackton\openclaw-main
$env:Path='d:\environment\nodejs24;' + $env:Path
$env:OPENCLAW_SKIP_CHANNELS='1'
node scripts/run-node.mjs --dev gateway
```

首次启动会触发一次构建，耗时会明显更长，属于正常现象。

## 7. 验证服务是否启动成功

新开一个 PowerShell 窗口执行：

```powershell
Invoke-WebRequest -UseBasicParsing -Uri "http://127.0.0.1:19001/health" | Select-Object -ExpandProperty Content
```

预期返回：

```json
{"ok":true,"status":"live"}
```

可选排查端口：

```powershell
Get-NetTCPConnection -State Listen | Where-Object { $_.LocalPort -in 19001,19003 } | Select-Object LocalAddress,LocalPort,OwningProcess
```

本次实测：

- `127.0.0.1:19001` 可用（health 正常）。
- `127.0.0.1:19003` 访问会返回 401（需要鉴权，属正常）。

## 8. 停止服务

在运行 Gateway 的那个终端按 `Ctrl + C`。

## 9. 下次启动最短路径

以后只需要：

```powershell
Set-Location d:\aihackton\openclaw-main
$env:Path='d:\environment\nodejs24;' + $env:Path
$env:OPENCLAW_SKIP_CHANNELS='1'
node scripts/run-node.mjs --dev gateway
```

## 10. 常见问题与处理

### Q1: `pnpm` 命令找不到

- 用 `pnpm.cmd -v` 测试。
- 确认当前会话已设置：

```powershell
$env:Path='d:\environment\nodejs24;' + $env:Path
```

### Q2: 报 `Node.js v22.16+ is required`

- 当前会话仍在用旧 Node。
- 先执行：

```powershell
$env:Path='d:\environment\nodejs24;' + $env:Path
node -v
```

确保输出是 `v24.x` 再继续。

### Q3: `pnpm gateway:dev` 报 `OPENCLAW_SKIP_CHANNELS 不是内部或外部命令`

- 这是 Windows 与类 Unix 环境变量语法差异。
- 改用第 6 节的 PowerShell 写法启动。

### Q4: 首次启动很慢

- 首次会触发构建与后处理，几分钟属于正常。
- 看终端是否持续有构建日志（例如 tsdown / runtime-postbuild）。

---

维护建议：

- 这个仓库是活跃项目，建议定期：

```powershell
Set-Location d:\aihackton\openclaw-main
$env:Path='d:\environment\nodejs24;' + $env:Path
pnpm.cmd install
```

- 如果更新后行为异常，先清理并重装依赖（谨慎执行）：

```powershell
Set-Location d:\aihackton\openclaw-main
Remove-Item -Recurse -Force .\node_modules
pnpm.cmd install
```
