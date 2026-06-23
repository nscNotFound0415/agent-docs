# RESTORE — D:\Agent\ 完全手动恢复剧本

> 适用场景:**不想用** `agent-infra/bootstrap.ps1` 自动脚本的情况,或者 agent-infra 仓本身不可用。
> 预计耗时:30-60 分钟(全部手动操作)。
>
> 如果 agent-infra 仓可达,**优先用 bootstrap**(详见 README.md 的"路 A"):
> ```powershell
> gh auth login
> gh repo clone nscNotFound0415/agent-infra D:\Agent\infra
> .\D:\Agent\infra\bootstrap.ps1
> ```
> 本文档是 bootstrap 不可用时的**完全手动 fallback**。

---

## 前置条件

| 工具 | 用途 | 安装 |
|---|---|---|
| **Git for Windows** | clone / push / pull | https://git-scm.com/download/win |
| **GitHub CLI (`gh`)** | PRIVATE 仓认证 | https://cli.github.com |
| **PowerShell 5.1+** | Windows 自带 | 系统自带 |
| **GitHub 认证** | `gh auth login`,token 需 `repo` scope | 一次性 |

> 装完后,在 PowerShell 里 `gh --version` 验证。

---

## Step 1:登录 GitHub

```powershell
gh auth login
# 选 GitHub.com → HTTPS → Yes → Login with a web browser
# 浏览器开 github.com/login/device,输入 8 位码完成认证
```

验证:

```powershell
gh auth status
# 应该显示: Logged in to github.com account <your-name> (keyring)
```

---

## Step 2:创建 `D:\Agent\` 顶层

```powershell
# 确保 D:\ 存在
Test-Path D:\

# 创建顶层(如果没有)
New-Item -Path 'D:\Agent' -ItemType Directory -Force
```

---

## Step 3:Clone 系统层(`agent-infra`)

```powershell
cd D:\Agent
gh repo clone nscNotFound0415/agent-infra infra
```

验证:

```powershell
Test-Path D:\Agent\infra\.git        # True
Test-Path D:\Agent\infra\watcher\watcher.ps1   # True
```

---

## Step 4:Clone 应用层(`backup-projects`)

```powershell
cd D:\Agent
gh repo clone nscNotFound0415/backup-projects 本地应用
cd 本地应用
git checkout 办公笔记本分支 2>$null   # 如果需要切到本机分支
# 验证
git branch --show-current   # 应该是 办公笔记本分支
git status                  # 应该 clean
```

如果 `办公笔记本分支` 不存在(因为这是新机器),用:

```powershell
git checkout -b 办公笔记本分支
git push -u origin 办公笔记本分支
```

---

## Step 5:Clone 知识层(`backup-obsidian`)

```powershell
cd D:\Agent
gh repo clone nscNotFound0415/backup-obsidian obsidian
cd obsidian
git checkout 办公笔记本分支 2>$null
git branch --show-current   # 应该是 办公笔记本分支
git status                  # 应该 clean
```

---

## Step 6(可选):Clone 第三方 skills 仓

`antigravity-awesome-skills/` 是第三方 fork,如果你用得到:

```powershell
cd D:\Agent
git clone https://github.com/sickn33/antigravity-awesome-skills.git
```

(这一步是 public 仓,不需要 gh auth。)

---

## Step 7:配置 git 身份(每台机器一次)

```powershell
git config --global user.name  "Your Name"
git config --global user.email "your@email"
```

---

## Step 8:创建 per-machine watcher 配置

`agent-infra\watcher\watcher.config.ps1` **不**进 git(每台机器不同)。从 example 创建:

```powershell
Copy-Item D:\Agent\infra\watcher\watcher.config.ps1.example `
          D:\Agent\infra\watcher\watcher.config.ps1
notepad  D:\Agent\infra\watcher\watcher.config.ps1
```

**修改**:
- `MyBranch = "..."` — 这台机器的分支(本机 → `办公笔记本分支`;另一台 → `main`)
- `TheirBranch = "..."` — 对方的分支(反过来)

示例(本机办公笔记本):

```powershell
$targets = @(
    @{
        Path        = "D:\Agent\本地应用"
        Label       = "LocalApps"
        MyBranch    = "办公笔记本分支"
        TheirBranch = "main"
    },
    @{
        Path        = "D:\Agent\obsidian"
        Label       = "Obsidian"
        MyBranch    = "办公笔记本分支"
        TheirBranch = "main"
    }
)
```

⚠️ **保存为 UTF-8 with BOM**(否则 PowerShell 在中文 Windows 上解析中文路径会乱码)。
在 PowerShell 里:

```powershell
$utf8Bom = New-Object System.Text.UTF8Encoding $true
[System.IO.File]::WriteAllText(
    "D:\Agent\infra\watcher\watcher.config.ps1",
    (Get-Content "D:\Agent\infra\watcher\watcher.config.ps1" -Raw),
    $utf8Bom
)
```

---

## Step 9:注册自动备份计划任务

```powershell
cd D:\Agent\infra\watcher
.\watcher.ps1 -Install
```

这会:
1. 在 Windows 任务计划程序注册 `Agent-Backup-Watcher`(每 5 分钟)
2. 使用 `-WindowStyle Hidden` — **不弹窗**
3. 输出到 `D:\Agent\infra\watcher\watcher.log`(`Start-Transcript`)
4. 1 分钟后第一次跑

验证:

```powershell
.\watcher.ps1 -Status
# REGISTERED

.\watcher.ps1 -DryRun
# [LocalApps] already in sync with origin/main 等

Get-Content watcher.log -Tail 30
```

---

## Step 10(可选):视频解析环境

如果你用视频解析 subagent:

```powershell
cd D:\Agent\infra
.\setup-video-parse.cmd
```

装: Docker Desktop、ffmpeg、markmap-cli、Python deps。
幂等可重跑。

---

## Step 11(关键):验证全链路

| 检查 | 命令 | 期望 |
|---|---|---|
| 三个仓都在 | `Get-ChildItem D:\Agent -Directory` | `infra`, `本地应用`, `obsidian` |
| infra 仓连通 | `git -C D:\Agent\infra status` | 无 fatal error |
| 本地应用 仓连通 | `git -C D:\Agent\本地应用 status` | 无 fatal error |
| obsidian 仓连通 | `git -C D:\Agent\obsidian status` | 无 fatal error |
| 计划任务注册 | `.\D:\Agent\infra\watcher\watcher.ps1 -Status` | `REGISTERED` |
| 日志在写 | `Get-Content D:\Agent\infra\watcher\watcher.log -Tail 30` | 看到 `[LocalApps]` 等输出 |
| 知识层/应用层边界 | `Get-Content D:\Agent\obsidian\AGENTS.md` | 看到 `.vault-rules.md` 引用 |

---

## D:\Agent\ 下的临时目录(恢复后会缺失)

这些目录**故意**不在 git 里:

| 目录 | 大小 | 性质 | 重建方式 |
|---|---:|---|---|
| `.codegraph/` | ~125 MB | CodeGraph daemon 跑时生成,从源码重建 | 启动 CodeGraph daemon(`codegraph --watch`)自动生成 |
| `.omo/` | <1 KB | opencode session state | 重启 opencode |
| `.agents/` | <1 KB | opencode 配置占位 | 重启 opencode |
| `_inspect/` | ~1.4 MB | 旧 `本地应用` 历史快照 | 永久丢失(不影响功能) |

---

## Live Vault (`D:\obsidian\`) 单独处理

`D:\obsidian/` 是 **live vault**(Obsidian 打开这个),与 `D:\Agent\obsidian/`(备份)是两份。

如果 `agent-live-vault` 仓已存在:

```powershell
# Clone live vault
cd D:\
gh repo clone nscNotFound0415/agent-live-vault obsidian
```

如果还没建过这个仓,见 README.md "Live Vault" 一节。

---

## 灾难场景速查

| 场景 | 走哪步 |
|---|---|
| 换新机器 | 完整 Step 1-11 |
| 某个仓坏了 | 重跑该仓的 Step 4 / 5 |
| watcher 不工作了 | 重跑 Step 9(`-Install`) |
| agent-infra 整个挂了 | 本文档的 fallback 路径(没用 infra 仓) |
| plan task 被 Windows 删 | 重跑 Step 9 |
| Live vault(`D:\obsidian\`)丢了 | 单独处理(见上节) |

---

## 双向同步工作原理(简要)

每台机器在 GitHub 上有**自己的分支**:
- 机器 A "办公笔记本" → push `办公笔记本分支`
- 机器 B "另一台" → push `main`
- watcher 每 5 分钟:fetch → merge 对方分支 → commit 本地 → push 自己分支
- 冲突:merge --abort + 跳过 commit/push + 日志告警(手动解决)

详见 README.md "双向同步是怎么工作的" 一节。

---

## 完整命令清单(复制粘贴用)

```powershell
# === 在新 Windows 机器上,以管理员身份打开 PowerShell ===

# 前置(只一次)
# 1. 装 Git for Windows (https://git-scm.com/download/win)
# 2. 装 gh CLI (https://cli.github.com)
# 3. gh auth login

# === 手动恢复 ===
New-Item -Path 'D:\Agent' -ItemType Directory -Force
Set-Location D:\Agent

gh repo clone nscNotFound0415/agent-infra infra
gh repo clone nscNotFound0415/backup-projects 本地应用
gh repo clone nscNotFound0415/backup-obsidian obsidian

git config --global user.name "Your Name"
git config --global user.email "your@email"

Set-Location D:\Agent\本地应用
git checkout 办公笔记本分支

Set-Location D:\Agent\obsidian
git checkout 办公笔记本分支

# 创建 per-machine 配置(注意 UTF-8 BOM)
Copy-Item D:\Agent\infra\watcher\watcher.config.ps1.example D:\Agent\infra\watcher\watcher.config.ps1
# 编辑 watcher.config.ps1 设 MyBranch/TheirBranch
# (Notepad 保存时选 UTF-8 with BOM)

# 注册计划任务
Set-Location D:\Agent\infra\watcher
.\watcher.ps1 -Install

# 验证
.\watcher.ps1 -Status
.\watcher.ps1 -DryRun
Get-Content watcher.log -Tail 30

# === 完成 ===
```

---

## 许可

本文档采用 MIT 协议。
