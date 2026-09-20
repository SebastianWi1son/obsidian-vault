# Win10 接入 Develop 同步 — 交接文档（方案 B）

> 面向对象：在 **Windows 10** 里运行的 agent（它没有本对话的上下文，本文件需自包含）。
> 撰写方：Ubuntu 侧的 agent。撰写时间：2026-09-20。
> 执行方式：按顺序做完第 1~8 步，然后按第 4 节验收、按第 5 节禁区自检、按第 6 节回报。

---

## 0. 任务摘要（先看这张表）

在这台双系统机器的 **Windows 10** 上安装 Syncthing，把 `D:\Develop` 建成 `develop` 文件夹的**第三个同步节点**，内容从另一台笔记本 `wilson-fedora` 全量拉取。

| 参数 | 值 |
|---|---|
| 文件夹 ID（**必须一字不差**） | `develop` |
| Windows 侧文件夹路径 | `D:\Develop`（`D:` = 卷标 `DATA` 的 NTFS 分区，请在第 2 步确认盘符） |
| 文件夹类型 | 发送和接收（Send & Receive） |
| 对端设备名 | `wilson-fedora` |
| 对端设备 ID | `O23ZEYE-5F2SGPU-KVGWJJF-G34IB3T-JNBGJSR-NUZNEUH-EMXZ5ZZ-66DALQK` |
| 对端地址 | `tcp://192.168.50.1:22000, dynamic` |
| 本机有线网卡 IP | **静态** `192.168.50.2/24`，**网关留空、DNS 留空** |
| 预期数据量 | 约 2.7 GB / 约 2.7 万文件（不含被忽略项） |
| Web GUI | http://127.0.0.1:8384 |

**只加 `wilson-fedora` 这一个设备。不要添加 Ubuntu 侧的设备。**

---

## 1. 背景与前提（理解为什么这样做）

1. 这是一台**双系统主机**：
   - Windows 10 装在 NVMe 盘 `nvme0n1` 上（p3 = 214 GB 系统盘；**p5 = 卷标 `DATA` 的 NTFS 分区，约 785 GB，本任务的目标盘**）。
   - Ubuntu 装在 SATA 盘 `sda` 上（sda1 `/`、sda2 EFI、sda3 swap、**sda4 `/home`**）。
   - **同一时刻只有一个系统在运行**，两个系统永不同时在线。
2. Ubuntu 侧的现状：`/home/wilson/Develop`（ext4，2.8 GB / 30879 文件 / 7 个 git 仓库）已经通过 Syncthing（版本 v1.27.2，folder id `develop`，发送和接收）与笔记本 `wilson-fedora` 双向同步。
3. Windows **读不了 ext4**，所以不看、不碰 Ubuntu 那块盘。Windows 侧的 `D:\Develop` 是一份**独立的第二副本**，内容由 Syncthing 从 `wilson-fedora` 拉取。
4. 为什么不同时添加 Ubuntu 的设备：Ubuntu 和 Windows 同机不同系统，**永远不同时在线**，互相添加只会产生永远连不上的连接尝试。跨系统的数据一致靠 `wilson-fedora` 中转。
5. 网线是**直连** `wilson-fedora` 笔记本，没有 DHCP 服务器。因此 Windows 侧必须手工设静态 IP，否则走不了内网直连（只能退公网中继，很慢）。
6. `.stignore`（忽略规则）**不会在设备之间同步**，每个节点必须各自放一份。

---

## 2. 执行步骤

### 第 1 步：配置有线网卡静态 IP

以**管理员**身份打开 PowerShell：

```powershell
# 1) 找到有线网卡的真实名字（InterfaceAlias）
Get-NetAdapter | Format-Table Name, InterfaceDescription, Status, LinkSpeed, MacAddress

# 2) 记下有线网卡名字（下面以「以太网」为例，实际以第 1 条输出为准）
$if = "以太网"

# 3) 清掉该网卡上可能存在的 DHCP 地址，再设静态 IP；不要设网关和 DNS
Remove-NetIPAddress  -InterfaceAlias $if -Confirm:$false -ErrorAction SilentlyContinue
Remove-NetRoute      -InterfaceAlias $if -Confirm:$false -ErrorAction SilentlyContinue
New-NetIPAddress     -InterfaceAlias $if -IPAddress 192.168.50.2 -PrefixLength 24
Set-DnsClientServerAddress -InterfaceAlias $if -ResetServerAddresses

# 4) 把该网络类型设为「专用」（专用网络才允许 Syncthing 被入站访问）
Set-NetConnectionProfile -InterfaceAlias $if -NetworkCategory Private

# 5) 验证：应当 TcpTestSucceeded = True
Test-NetConnection 192.168.50.1 -Port 22000
Get-NetIPAddress -InterfaceAlias $if
```

- 无线网卡保持原样（上网用），不要改动。
- **不要**给有线网卡设置默认网关，否则会打乱上网路由。

### 第 2 步：确认数据盘，准备目录与忽略规则

```powershell
Get-Volume | Where-Object {$_.FileSystemLabel -eq 'DATA'} | Format-Table DriveLetter, FileSystemLabel, FileSystem, SizeRemaining, Size
Get-Partition | Format-Table DiskNumber, PartitionNumber, DriveLetter, Size
```

- 确认卷标 `DATA` 的盘符（预期 `D:`）。本步骤后续按 `D:\Develop` 处理；若盘符不同，全文替换。
- 创建目录：

```powershell
if (Test-Path 'D:\Develop') {
    $n = (Get-ChildItem 'D:\Develop' -Force -ErrorAction SilentlyContinue | Measure-Object).Count
    Write-Host "D:\Develop 已存在，条目数 = $n"
    # 若条目数 > 0 且不是空的 .stfolder：停下来上报，不要删除、不要覆盖
} else {
    New-Item -ItemType Directory -Path 'D:\Develop' | Out-Null
    Write-Host "D:\Develop 已创建"
}
```

- **在首次同步之前**写入忽略规则文件 `D:\Develop\.stignore`（UTF-8 无 BOM，内容逐字如下，注意 `//` 是注释、每行一条规则）：

```
// 三方必须保持一致：Ubuntu / Fedora / Windows
.venv/
node_modules/
__pycache__/
*.pyc
*.pyo
.pytest_cache/
.mypy_cache/
.ruff_cache/
.DS_Store
Thumbs.db
desktop.ini
```

PowerShell 写法（避免 BOM 与编码问题）：

```powershell
$ignore = @'
// 三方必须保持一致：Ubuntu / Fedora / Windows
.venv/
node_modules/
__pycache__/
*.pyc
*.pyo
.pytest_cache/
.mypy_cache/
.ruff_cache/
.DS_Store
Thumbs.db
desktop.ini
'@
[System.IO.File]::WriteAllText('D:\Develop\.stignore', $ignore, (New-Object System.Text.UTF8Encoding($false)))
Get-Content 'D:\Develop\.stignore'
```

> 为什么必须忽略 `.venv/`：`Tools/openocd-mcp/.venv` 里有 3173 个小文件，且包含 Linux 符号链接（如 `.venv/bin/python` → `python3`），在 Windows 上无意义且拖慢同步。

### 第 3 步：安装 Syncthing（要开机自启）

官方没有 Windows 安装器，以下两条路任选，**优先第一条**：

**A. Syncthing Windows Setup（有安装器，支持"所有用户 + Windows 服务"）**

```powershell
winget search syncthing
winget install --id Syncthing.Syncthing      # 若无此包，从 https://github.com/Bill-Stewart/SyncthingWindowsSetup/releases 下载安装
```

安装选项建议：安装给当前用户（默认）+ 勾选「登录时启动」；或安装给所有用户以注册为 Windows 服务。

**B. SyncTrayzor（托盘 GUI，省心）**

```powershell
winget install --id SyncTrayzor.SyncTrayzor
```

要求与记录项：

- 版本：优先 **v1.x（1.30.x）**，与 Ubuntu 侧同一大版本最稳妥。若只有 v2.x 可用也可以（v2 与 v1 协议兼容），但**请记录实际版本号**以便回报。
- 必须在**开机/登录后自动启动**。
- 记录配置文件目录（常见 `%LOCALAPPDATA%\Syncthing` 或 `%APPDATA%\Syncthing`）与可执行文件路径。
- 打开 GUI：`http://127.0.0.1:8384`，设置管理员账号密码（可选但建议）。
- 不需要开启「匿名使用统计」。

### 第 4 步：添加对端设备

GUI → **操作 → 显示 ID** 记下本机（Windows）设备 ID（第 7 步要用）。

GUI → **添加远程设备**：

| 字段 | 值 |
|---|---|
| 设备 ID | `O23ZEYE-5F2SGPU-KVGWJJF-G34IB3T-JNBGJSR-NUZNEUH-EMXZ5ZZ-66DALQK` |
| 设备名 | `wilson-fedora` |
| 地址 | `tcp://192.168.50.1:22000, dynamic` |
| 压缩 | 默认（全部数据） |

保存后到「远程设备」列表确认 `wilson-fedora` 的状态变为 **已连接（Connected）**。

- 若一直显示「已断开」：回到第 1 步核对 IP，并检查 Windows 防火墙是否允许 `syncthing.exe` 在**专用网络**上通信（安装程序通常会加规则；否则在「Windows 防火墙 → 允许应用通过防火墙」里手动允许，或执行两次 `Test-NetConnection` 复测）。

### 第 5 步：创建文件夹并共享

GUI → **添加文件夹**：

| 字段 | 值 |
|---|---|
| 文件夹 ID | `develop`（**必须与 Ubuntu/Fedora 完全一致，区分大小写**） |
| 文件夹标签 | `Develop` |
| 文件夹路径 | `D:\Develop` |
| 文件夹类型 | **发送和接收** |
| 文件版本控制 | 无（三方当前都是"无"，不要开） |
| 共享给 | 勾选 `wilson-fedora` |

保存后应看到文件夹状态为「正在同步 / 扫描中」，随后进入「已同步（Up to Date）」。

- 若报「文件夹标记缺失（folder marker missing）」：确认 `D:\Develop\.stfolder` 是否存在，不存在则手工新建空目录（不要手工删它）。
- 若 `D:\Develop` 里本来就有不该有的东西，先停下来报告。

### 第 6 步：等待首次全量同步

- 预期从 `wilson-fedora` 拉取约 **2.7 GB / 约 2.7 万文件**（排除 `.venv`、`node_modules`、`__pycache__` 等之后）。走千兆直连，通常几分钟内完成。
- 期间不要手工改动 `D:\Develop` 里的文件。

### 第 7 步：需要**人类**在 Fedora 笔记本上配合的动作（agent 无法代办，必须明确提示用户）

Windows 端在第 5 步发出共享后，Fedora 侧 Syncthing 会弹窗提示 「`wilson-win` 想要共享文件夹 `develop`」。请用户在该笔记本上：

1. **接受**该共享请求；
2. 弹窗中若询问使用哪个本地路径，**选择已有的 `develop` 文件夹**（Fedora 上早已存在同一 ID 的文件夹），**不要新建第三个路径**；
3. 在 Fedora 的 `develop` 文件夹 → 「共享」选项卡里确认已勾选 `wilson-win`。

如果 Fedora 侧始终没有弹窗，则让用户在 Fedora 上：添加远程设备（粘贴本机 Windows 设备 ID，即第 4 步记下的那个）→ 在 `develop` 文件夹的共享列表里勾选该设备。

### 第 8 步：确认同步完成

见下一节验收清单。

---

## 3. 验收清单（逐条给出证据）

1. `Test-NetConnection 192.168.50.1 -Port 22000` → `TcpTestSucceeded : True`
2. `Get-NetIPAddress -InterfaceAlias <有线网卡>` → `192.168.50.2/24`，且该接口**没有**默认网关
3. GUI「远程设备」→ `wilson-fedora` 状态 = **已连接**
4. GUI「文件夹」→ `develop` 状态 = **已同步（Up to Date）**，无失败项（Failed Items = 0）
5. `D:\Develop` 下应有 6 个顶层目录：`Archive`、`Documents`、`Notes`、`Projects`、`Tools`、`Workspace`
6. `D:\Develop\Tools\openocd-mcp\.venv` **不存在**（证明忽略规则生效）
7. `D:\Develop\.stignore` 存在且内容与第 2 步一致；`D:\Develop\.stfolder` 存在
8. 数量核对（PowerShell）：

```powershell
$all = Get-ChildItem 'D:\Develop' -Recurse -Force -File -ErrorAction SilentlyContinue
Write-Host ("文件数 = {0}" -f $all.Count)
Write-Host ("总大小 = {0:N2} GB" -f (($all | Measure-Object Length -Sum).Sum / 1GB))
# 期望：文件数 ≈ 27000（2.6 万~2.9 万之间都可接受），总大小 ≈ 2.7 GB
```

9. 用 Syncthing REST API 交叉验证（GUI → 操作 → 设置 → 图形用户界面 → API 密钥）：

```powershell
$key = "<粘贴 API 密钥>"
$h = @{ 'X-API-Key' = $key }
Invoke-RestMethod -Headers $h "http://127.0.0.1:8384/rest/db/status?folder=develop" |
    Select-Object state, errors, needFiles, needBytes, globalFiles, globalBytes, localFiles
# 期望：state = idle, errors = 0, needFiles = 0
```

---

## 4. 禁区（这些事明确不要做）

- ❌ 不要挂载、格式化、写入或"修复" `sda` 上的任何分区（那是 Ubuntu 的系统盘）；不要安装 Ext2Fsd / Paragon 之类读写 ext4 的驱动。
- ❌ 不要在 `C:` 上另建一份 Develop 副本；数据只放 `DATA` 分区。
- ❌ 不要把 Ubuntu 侧的 Syncthing 设备 ID 加进来（它永不与 Windows 同时在线）。
- ❌ 不要把文件夹类型设为「仅发送」或「仅接收」；不要开启文件版本控制。
- ❌ 不要手工删除 `D:\Develop\.stfolder`、`.stignore`，不要在同步过程中手工增删文件。
- ❌ 若 `D:\Develop` 已存在且非空（且非本任务产物），**停下来报告**，不要覆盖或删除。
- ❌ 不要改动 Windows 快速启动/休眠设置（方案 B 下 Ubuntu 完全不碰 NTFS，无需改动）。
- ❌ 不要尝试读取或迁移 Ubuntu 侧 `/home/wilson/Develop` 的现有数据——内容全部通过网络从 `wilson-fedora` 拉取。

---

## 5. 回报给用户的内容（做完后按此清单汇报）

1. Syncthing 版本号 + 安装方式（A/B）+ 配置文件目录 + 可执行文件路径
2. 有线网卡名 + `Get-NetIPAddress` 输出（确认 192.168.50.2/24、无网关）
3. `Test-NetConnection 192.168.50.1 -Port 22000` 结果
4. **本机 Windows 设备 ID**（第 7 步人类在 Fedora 侧需要它）
5. 设备 `wilson-fedora` 的连接状态
6. `develop` 文件夹状态 + 第 3 节第 8、9 条的数量核对结果
7. 任何报错原文（不要只写"失败了"）
8. 若中途卡在第 7 步（Fedora 侧未接受共享），明确说明"等待用户在 Fedora 上接受共享"

---

## 附录 A：可选的全自动配置路径（用 `syncthing cli` 代替 GUI 点击）

在 Syncthing 已启动的前提下，可用命令行完成第 4、5 步（**语法随版本略有差异；任一条报错就回退到 GUI 操作，不要硬试**）：

```powershell
& "<syncthing.exe 路径>" cli config devices add --device-id O23ZEYE-5F2SGPU-KVGWJJF-G34IB3T-JNBGJSR-NUZNEUH-EMXZ5ZZ-66DALQK --name wilson-fedora
& "<syncthing.exe 路径>" cli config devices wilson-fedora addresses add tcp://192.168.50.1:22000
& "<syncthing.exe 路径>" cli config devices wilson-fedora addresses add dynamic
& "<syncthing.exe 路径>" cli config folders add --id develop --label Develop --path 'D:\Develop' --type sendreceive
& "<syncthing.exe 路径>" cli config folders develop devices add --device-id O23ZEYE-5F2SGPU-KVGWJJF-G34IB3T-JNBGJSR-NUZNEUH-EMXZ5ZZ-66DALQK
```

完成后**必须**回到 GUI 或 REST API 复核（第 3 节第 3~5 条）。

## 附录 B：参考事实（Ubuntu 侧现状，供排查用）

| 项 | 值 |
|---|---|
| Ubuntu Syncthing 版本 | v1.27.2（Debian 包） |
| Ubuntu 文件夹配置 | `id=develop`，`path=/home/wilson/Develop`，`type=sendreceive`，`rescanIntervalS=3600`，`fsWatcherEnabled=true` |
| Ubuntu 设备名 | `wilson-ubuntu` |
| Fedora 设备名 / 地址 | `wilson-fedora` / `tcp://192.168.50.1:22000` |
| Ubuntu 侧有线 IP | 192.168.50.2/24（与 Windows 复用同一网口同一地址，两者不同时在线，不冲突） |
| 原始数据规模 | 2.8 GB / 30879 文件 / 3 个 `.venv` 路径 / 7 个 git 仓库 |
