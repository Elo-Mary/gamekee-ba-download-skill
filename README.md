# gamekee-ba-download-skill

![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)
![OpenCode Skill](https://img.shields.io/badge/OpenCode-Skill-blue)

> 中文 ｜ [English](./README.en.md)

一个 [OpenCode](https://opencode.ai) skill：让 AI agent 按既定流程，从 gamekee [碧蓝档案（BA）图鉴](https://www.gamekee.com/ba/) 批量下载角色图片。

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

- **[OpenCode](https://opencode.ai)** 已安装。
- **Playwright MCP**（提供 `browser_navigate` / `browser_run_code_unsafe` 等浏览器工具，本 skill 的抓取阶段依赖它们）。注意：基础版 OpenCode **不**自带 Playwright MCP，需二选一：
  - **推荐**：安装 [oh-my-openagent](https://github.com/code-yeongyu/oh-my-openagent) 插件，它会自动配置 Playwright MCP，开箱即用；或
  - 手动在 `opencode.json` 中添加 MCP 服务器：
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
  - 两者均需 **Node.js 18+**；Playwright 会在首次使用时自动下载浏览器二进制。
- **PowerShell**（下载阶段使用 `.ps1` 脚本）：Windows 自带的 PowerShell 5.1 即可，其他系统需安装 [PowerShell Core (pwsh)](https://learn.microsoft.com/powershell/scripting/install/installing-powershell)。

## 安装

> OpenCode skill 就是一个放在 skills 目录下的 `SKILL.md` 文件。以下两种方式任选其一。

### 方式一：让 agent 帮你装（推荐）

把下面这段话直接发给你的 OpenCode agent：

```
帮我安装 OpenCode skill「gamekee-ba-download」（批量下载 gamekee 碧蓝档案角色图片）：
1. 从 GitHub 仓库 https://github.com/Elo-Mary/gamekee-ba-download-skill 获取 SKILL.md（克隆仓库或直接下载该文件均可）
2. 把 SKILL.md 放到 OpenCode 的 skills 目录下，路径为 gamekee-ba-download/SKILL.md（目录不存在就创建）。
   目录位置：Linux/macOS 为 ~/.config/opencode/skills/，Windows 为 %USERPROFILE%\.config\opencode\skills\
3. 完成后确认文件已就位
```

### 方式二：git clone + 手动拷贝

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

## 使用

在 OpenCode 会话里直接用自然语言触发，例如：

- 「帮我下载碧蓝档案回忆大厅的图片到 `D:\BA图`」
- 「下载 gamekee BA 图鉴的官方介绍图」
- 「把回忆大厅和官方介绍都下了」

可指定参数（有默认值，不填则用默认）：

| 参数 | 默认值 | 说明 |
|---|---|---|
| `target` | `hydt` | `hydt`=回忆大厅 ｜ `gfjs`=官方介绍 ｜ `both`=两者都下 |
| `output_dir` | 当前工作目录 | 图片落盘根目录 |
| `list_url` | `https://www.gamekee.com/ba/second/23941` | 角色花名册入口（实装学生） |

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
- **更多服务器**：国际服 / 国服 / 繁中服