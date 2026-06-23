# D:\Agent\ — 系统架构与恢复文档(PUBLIC)

> 这是 `nscNotFound0415` 个人多机器知识/代码工作流的**公开文档入口**。
> 完整脚本与配置在私有仓 `nscNotFound0415/agent-infra`。

---

## 这是什么

`D:\Agent\` 是一个跨多台 Windows 机器可同步的工作目录结构,包含:

- **知识层**:Obsidian vault(笔记、RAG 索引、教学区)
- **应用层**:AI 应用代码(视频解析、Bilibili 摘要等 subagent)
- **系统层**:bootstrap + watcher + 灾难恢复脚本

整个系统通过 Git + GitHub 实现**多机器双向同步**,任何一个仓库出问题都能从 GitHub 完整恢复。

---

## 仓库结构与可见性

| 仓 | 用途 | 大小 | 可见性 |
|---|---|---|---|
| `nscNotFound0415/agent-docs` | **本仓**:公开文档 | ~30 KB | ✅ **PUBLIC** |
| `nscNotFound0415/agent-infra` | bootstrap / watcher / 脚本 | ~12 KB | 🔒 PRIVATE |
| `nscNotFound0415/backup-obsidian` | vault 笔记备份 | ~350 KB+ | 🔒 PRIVATE |
| `nscNotFound0415/backup-projects` | AI 应用代码 | ~130-180 MB | 🔒 PRIVATE |

> `agent-infra` 是 PRIVATE,因为脚本涉及机器身份 + 同步配置。
> 个人内容(笔记、代码)也都 PRIVATE。
> 只有这个文档仓是 PUBLIC,作为恢复流程的入口。

---

## 恢复路径(两条)

### 路 A:自动(推荐,如果 GitHub 可达)

```powershell
# 前置:装 gh CLI + 登录
gh auth login

# 主流程
gh repo clone nscNotFound0415/agent-infra D:\Agent\infra
.\D:\Agent\infra\bootstrap.ps1
```

`bootstrap.ps1` 会:
1. 探测机器身份(hostname / env var)
2. 拉 `backup-obsidian` 和 `backup-projects` 两个 PRIVATE 仓
3. 生成 per-machine watcher 配置
4. 注册 Windows 计划任务(无窗口,5 分钟双向同步)

### 路 B:完全手动(agent-infra 不可达时)

适合:agent-infra 仓坏了 / 不想用脚本 / 想理解每一步在做什么。

详细:见 **[RESTORE.md](./RESTORE.md)**

---

## D:\Agent\ 顶层目录(恢复后应有的样子)

```
D:\Agent\
├── infra/                          ← agent-infra 仓(clone 得到)
├── 本地应用\                       ← backup-projects 仓(clone 得到)
├── obsidian/                       ← backup-obsidian 仓(clone 得到)
├── antigravity-awesome-skills/    ← 第三方 fork(独立 clone)
├── .codegraph/                     ← 不进 git,daemon 启动时自动重建
├── .omo/ .agents/                  ← 不进 git,临时状态,可丢弃
└── _inspect/                       ← 不进 git,历史快照,可丢弃
```

不在 git 里的目录恢复后会缺失(都是可重建或可丢弃的)。详见 RESTORE.md 末节。

---

## 关键文档

- **[RESTORE.md](./RESTORE.md)** — 完整手动恢复剧本(15-30 分钟)
- 私有仓 [agent-infra](https://github.com/nscNotFound0415/agent-infra) — 脚本 + watcher
- 私有仓 [backup-obsidian](https://github.com/nscNotFound0415/backup-obsidian) — vault 备份
- 私有仓 [backup-projects](https://github.com/nscNotFound0415/backup-projects) — 应用代码

---

## 双向同步是怎么工作的

```
[机器 A "办公笔记本"]
        ↓
   push: 办公笔记本分支
        ↓
   ┌─── GitHub ───┐
   │  协调两台机  │
   └──────────────┘
        ↓
   pull: main
        ↓
[机器 B "另一台"]
```

- 每台机器在 GitHub 上有自己的分支(`main` 或 `办公笔记本分支`)
- `D:\Agent\infra\watcher\watcher.ps1` 每 5 分钟跑一次:
  1. `git fetch origin`
  2. `git merge origin/<对方的分支>` — 拉对方的改动
  3. `git add -A && git commit` — 提交本地改动
  4. `git push origin <自己的分支>` — 推上去
- 冲突:`merge --abort` + 日志告警 + 跳过 commit/push,需手动解决

详见 `agent-infra/RESTORE.md`(PRIVATE,但你已登录后能看)。

---

## 验证恢复成功的快速清单

```
[ ] gh auth status                → 已登录
[ ] ls D:\Agent\infra            → 有 .git/
[ ] ls D:\Agent\obsidian          → 有 .git/
[ ] ls D:\Agent\本地应用          → 有 .git/
[ ] .\D:\Agent\infra\watcher\watcher.ps1 -Status  → REGISTERED
[ ] Get-Content D:\Agent\infra\watcher\watcher.log -Tail 30  → 最近一次运行日志
[ ] (可选)打开 Obsidian 加载 D:\obsidian\  → 笔记可见
```

---

## 灾难场景速查

| 场景 | 走哪条 |
|---|---|
| 换新机器 | 路 A(自动)或 RESTORE.md(手动) |
| 某个仓坏了 | `cd D:\Agent\<仓>; git pull` |
| watcher 不工作了 | `cd D:\Agent\infra\watcher; .\watcher.ps1 -Status` 看状态,有问题重跑 `-Install` |
| agent-infra 仓整个挂了 | 从本仓的 RESTORE.md 走"全手动"路径(不依赖 infra 仓) |
| 计划任务被 Windows 重置删了 | 重跑 `.\watcher.ps1 -Install` |
| 误删了 .codegraph 等临时目录 | **正常**,daemon 启动会重建 |

---

## 许可与隐私

本仓内容(本 README + RESTORE.md)采用 MIT 协议,可自由分享。

其他三个仓(agent-infra、backup-obsidian、backup-projects)是 PRIVATE,包含个人笔记、代码、API 配置等敏感内容,请勿外传。
