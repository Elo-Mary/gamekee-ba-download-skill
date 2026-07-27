# gamekee-ba-download-skill

![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)
![Agent Skills](https://img.shields.io/badge/Agent%20Skills-standard-blue)

> 中文 ｜ [English](./README.en.md)

一个 [Agent Skills](https://agentskills.io) 标准技能：让 AI agent 按既定流程，从 gamekee [碧蓝档案（BA）图鉴](https://www.gamekee.com/ba/) 批量下载角色图片。

> 当前仅支持 **日服** 的 **回忆大厅** 与 **官方介绍** 两种图。完整操作手册见 [`SKILL.md`](./SKILL.md)。

## 它能做什么

- 自动抓取 gamekee「实装学生」图鉴花名册（约 267 个角色）。
- 逐角色打开详情页，按指定类型抽取图片 URL。
- 用 PowerShell 批量下载、按真实文件头判定格式落盘，支持断点续传与失败重试。
- 输出按图片类型分子文件夹归档：

```
output_dir/
├── 回忆大厅/
│   ├── 日奈.png
│   ├── 瞬（幼女）.png
│   └── …
├── 官方介绍/
│   └── …
└── batches/            # 抽取的 URL 清单（中间产物）
    ├── batch01.json
    └── …
```

## 前置要求

- **Playwright MCP**（提供 `browser_navigate` / `browser_run_code_unsafe` 等浏览器工具，本 skill 的抓取阶段依赖它们）。任何支持 MCP 的 agent 都需要此工具——基础版 agent **不**自带 Playwright MCP，需手动添加。
  - **便捷方式**：安装 [oh-my-openagent](https://github.com/code-yeongyu/oh-my-openagent) 插件可自动配置（适用于 OpenCode）；或
  - 在对应平台的 MCP 配置文件中添加以下服务器定义（配置文件名因平台而异：OpenCode 使用 `opencode.json`，Claude Code 使用 `.mcp.json`，Goose 使用 `config.yaml`——请查阅各平台的 MCP 文档）：
    
    ```json
    {
      "mcp": {
        "playwright": {
          "type": "local",
          "command": ["npx", "@playwright/mcp@latest"]
        }
      }
    }
    ```
  - 需 **Node.js 18+**；Playwright 会在首次使用时自动下载浏览器二进制。
- **PowerShell**（下载阶段使用 `.ps1` 脚本）：Windows 自带的 PowerShell 5.1 即可，其他系统需安装 [PowerShell Core (pwsh)](https://learn.microsoft.com/powershell/scripting/install/installing-powershell)。

## 安装

> Agent Skill 就是一个放在对应平台 skills 目录下的 `SKILL.md` 文件。

### 让 Agent 帮你安装（推荐）

任何平台的 agent 都应该有能力自行找到 skill 存储目录并完成安装。把下面这段话发给你的 agent：

```
帮我安装 Agent Skill「gamekee-ba-download」（批量下载 gamekee 碧蓝档案角色图片）：
1. 从 GitHub 仓库 https://github.com/Elo-Mary/gamekee-ba-download-skill 获取 SKILL.md（克隆仓库或直接下载该文件均可）
2. 找出你当前所在平台的 Agent Skills 存储目录（即该平台加载 skill 的路径），把 SKILL.md 放到 gamekee-ba-download/SKILL.md（目录不存在就创建）
3. 完成后确认文件已就位
```

### 手动安装

以下按平台列举手动拷贝步骤。如果你的平台未列出，见末尾的「其它平台」。

<details>
<summary>OpenCode</summary>

**Skill 路径**：`~/.config/opencode/skills/gamekee-ba-download/SKILL.md`

```bash
git clone https://github.com/Elo-Mary/gamekee-ba-download-skill.git
```

Linux / macOS：

```bash
mkdir -p ~/.config/opencode/skills/gamekee-ba-download
cp gamekee-ba-download-skill/SKILL.md ~/.config/opencode/skills/gamekee-ba-download/SKILL.md
```

Windows (PowerShell)：

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.config\opencode\skills\gamekee-ba-download"
Copy-Item ".\gamekee-ba-download-skill\SKILL.md" "$env:USERPROFILE\.config\opencode\skills\gamekee-ba-download\SKILL.md"
```

安装完成后 **新开一个 OpenCode 会话** 即可生效。

</details>

<details>
<summary>Claude Code</summary>

**Skill 路径**：`~/.claude/skills/gamekee-ba-download/SKILL.md`

> 根据 [Claude Code 官方文档](https://code.claude.com/docs/en/skills)，Claude Code 从 `~/.claude/skills/` 目录加载 skill。如果官方文档已更新，请以最新文档为准。

```bash
git clone https://github.com/Elo-Mary/gamekee-ba-download-skill.git
```

Linux / macOS：

```bash
mkdir -p ~/.claude/skills/gamekee-ba-download
cp gamekee-ba-download-skill/SKILL.md ~/.claude/skills/gamekee-ba-download/SKILL.md
```

Windows (PowerShell)：

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.claude\skills\gamekee-ba-download"
Copy-Item ".\gamekee-ba-download-skill\SKILL.md" "$env:USERPROFILE\.claude\skills\gamekee-ba-download\SKILL.md"
```

安装完成后 **新开一个 Claude Code 会话** 即可生效。

</details>

<details>
<summary>Goose</summary>

**Skill 路径**：`~/.agents/skills/gamekee-ba-download/SKILL.md`

> 根据 [Agent Skills 标准](https://agentskills.io) 及 Goose 文档，Goose 从 `~/.agents/skills/` 目录加载 skill。如果 Goose 官方文档已更新，请以最新文档为准。

```bash
git clone https://github.com/Elo-Mary/gamekee-ba-download-skill.git
```

Linux / macOS：

```bash
mkdir -p ~/.agents/skills/gamekee-ba-download
cp gamekee-ba-download-skill/SKILL.md ~/.agents/skills/gamekee-ba-download/SKILL.md
```

Windows (PowerShell)：

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.agents\skills\gamekee-ba-download"
Copy-Item ".\gamekee-ba-download-skill\SKILL.md" "$env:USERPROFILE\.agents\skills\gamekee-ba-download\SKILL.md"
```

安装完成后 **新开一个 Goose 会话** 即可生效。

</details>

<details>
<summary>Cursor</summary>

**Skill 路径**：`.cursor/skills/gamekee-ba-download/SKILL.md`

> Cursor 的 skills 目录通常为项目内的 `.cursor/skills/`。请查阅 [Cursor 官方文档](https://docs.cursor.com) 确认最新路径。以下为参考步骤：

```bash
git clone https://github.com/Elo-Mary/gamekee-ba-download-skill.git
```

Linux / macOS：

```bash
mkdir -p .cursor/skills/gamekee-ba-download
cp gamekee-ba-download-skill/SKILL.md .cursor/skills/gamekee-ba-download/SKILL.md
```

Windows (PowerShell)：

```powershell
New-Item -ItemType Directory -Force -Path ".cursor\skills\gamekee-ba-download"
Copy-Item ".\gamekee-ba-download-skill\SKILL.md" ".cursor\skills\gamekee-ba-download\SKILL.md"
```

安装完成后在 Cursor 中重启或重新加载即可生效。

</details>

<details>
<summary>其它平台</summary>

由于 agent 平台众多，无法逐一列举。以下两种方式任选其一：

**方式一：让 Agent 帮你安装（推荐）**

把上方「让 Agent 帮你安装」中的提示词发给你的 agent。agent 会自行找到当前平台的 skill 存储目录并完成安装。

**方式二：手动安装**

1. 让 agent 帮你找出当前平台的 Agent Skills 存储路径；或自行查阅该平台的官方文档找到该路径。
2. 从 GitHub 仓库 https://github.com/Elo-Mary/gamekee-ba-download-skill 获取 `SKILL.md`（克隆仓库或直接下载该文件）。
3. 将 `SKILL.md` 放到该平台的 skills 目录下，路径为 `gamekee-ba-download/SKILL.md`（目录不存在就创建）。
4. 安装完成后重启或新开会话即可生效。

</details>

## 平台支持

| 平台          | 支持状态                           |
| ----------- | ------------------------------ |
| OpenCode    | 支持                             |
| Claude Code | 支持                             |
| Goose       | 支持                             |
| Cursor      | 支持                             |
| Codex CLI   | 暂不支持（需 AGENTS.md 嵌入或插件打包，留待未来） |

## 使用

在你的 agent 会话里直接用自然语言触发，例如：

- 「帮我下载碧蓝档案回忆大厅的图片到 `D:\BA图`」
- 「下载 gamekee BA 图鉴的官方介绍图」
- 「把回忆大厅和官方介绍都下了」

可指定参数（有默认值，不填则用默认）：

| 参数           | 默认值                                       | 说明                                      |
| ------------ | ----------------------------------------- | --------------------------------------- |
| `target`     | `hydt`                                    | `hydt`=回忆大厅 ｜ `gfjs`=官方介绍 ｜ `both`=两者都下 |
| `output_dir` | 当前工作目录                                    | 图片落盘根目录                                 |
| `list_url`   | `https://www.gamekee.com/ba/second/23941` | 角色花名册入口（实装学生）                           |

agent 会自行走完「抓花名册 → 逐角色抽图 → 下载 → 校验重试」全流程，中途可断点续传、可重跑补下失败项。详见 [`SKILL.md`](./SKILL.md)。

## 工作原理（简述）

1. 浏览器打开 gamekee 实装学生图鉴，抽取全部角色 ID。
2. 每 12 个一批，打开角色详情页，按 `target` 选择对应选择器读取 `img.src`。
3. 抽到的 URL 写入 `batches/batchNN.json`。
4. PowerShell 下载每张图，**按下载后的真实文件头判定格式**（CDN 会做内容协商，URL 后缀不可信），落到对应子文件夹。
5. 完整性校验 + 对失败/未渲染项统一重试。

> gamekee 的 content JSON 接口被 Tencent EdgeOne WAF 拦截，因此本 skill 走「浏览器读 `img.src` → PowerShell 下载」路线，不走 JSON API 捷径。完整约束与故障排查见 [`SKILL.md`](./SKILL.md)。

## 当前支持范围

- **游戏**：蔚蓝档案（Blue Archive）
- **区服**：日服
- **图片类型**：回忆大厅、官方介绍
- **来源**：gamekee.com 碧蓝档案图鉴 · 实装学生（约 267 名角色，含联动角色）

## Roadmap

- **更多图片类型**：设定集（日文 / 繁中）、本家画、表情等（需走图集模式，按 `.header-container` 文本切块取该节区全部图）
- **更多服务器**：国际服 / 国服
