# gamekee-ba-download-skill

![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)
![Agent Skills](https://img.shields.io/badge/Agent%20Skills-standard-blue)

> English ｜ [中文](./README.md)

An [Agent Skills](https://agentskills.io)-standard skill: lets an AI agent batch-download character images from the [Blue Archive (BA) wiki](https://www.gamekee.com/ba/) on gamekee, following a fixed pipeline.

> Currently supports only the **JP server** and two image types: **Memorial Lobby** and **Official Introduction**. Full operation manual in [`SKILL.md`](./SKILL.md).

## What It Does

- Automatically scrapes the gamekee "Implemented Students" roster (~267 characters).
- Opens each character's detail page and extracts image URLs by the specified type.
- Downloads images in bulk via PowerShell, determines format from real file headers (CDN content negotiation makes URL extensions unreliable), with checkpoint resume and failure retry.
- Organizes output into subdirectories by image type:

```
output_dir/
├── 回忆大厅/
│   ├── 日奈.png
│   ├── 瞬（幼女）.png
│   └── …
├── 官方介绍/
│   └── …
└── batches/            # extracted URL manifests (intermediate artifacts)
    ├── batch01.json
    └── …
```

## Prerequisites

- **Playwright MCP** (provides `browser_navigate` / `browser_run_code_unsafe` and other browser tools; the scraping phase depends on them). Any MCP-supporting agent needs this tool — base agents do **not** bundle Playwright MCP and require manual configuration.
  - **Convenient way**: install the [oh-my-openagent](https://github.com/code-yeongyu/oh-my-openagent) plugin for automatic configuration (OpenCode); or
  - Add the MCP server definition to your platform's config file (config file name varies by platform: OpenCode uses `opencode.json`, Claude Code uses `.mcp.json`, Goose uses `config.yaml` — consult each platform's MCP docs):
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
  - Requires **Node.js 18+**; Playwright downloads browser binaries automatically on first use.
- **PowerShell** (used by the `.ps1` download script): Windows ships with PowerShell 5.1; other platforms need [PowerShell Core (pwsh)](https://learn.microsoft.com/powershell/scripting/install/installing-powershell).

## Installation

> An Agent Skill is simply a `SKILL.md` file placed in the corresponding platform's skills directory. Installation steps vary by platform below.

<details>
<summary>OpenCode</summary>

**Skill Path**: `~/.config/opencode/skills/gamekee-ba-download/SKILL.md`

**Method 1: Let the agent install it (recommended)**

Paste the following to your OpenCode agent:

```
Install the OpenCode skill "gamekee-ba-download" (batch-download Blue Archive character images from gamekee):
1. Get SKILL.md from the GitHub repo https://github.com/Elo-Mary/gamekee-ba-download-skill (clone the repo or download the file directly).
2. Place SKILL.md in the OpenCode skills directory at gamekee-ba-download/SKILL.md (create the directory if needed).
   Path: Linux/macOS ~/.config/opencode/skills/, Windows %USERPROFILE%\.config\opencode\skills\
3. Confirm the file is in place when done.
```

**Method 2: Manual copy**

```bash
git clone https://github.com/Elo-Mary/gamekee-ba-download-skill.git
```

Linux / macOS:

```bash
mkdir -p ~/.config/opencode/skills/gamekee-ba-download
cp gamekee-ba-download-skill/SKILL.md ~/.config/opencode/skills/gamekee-ba-download/SKILL.md
```

Windows (PowerShell):

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.config\opencode\skills\gamekee-ba-download"
Copy-Item ".\gamekee-ba-download-skill\SKILL.md" "$env:USERPROFILE\.config\opencode\skills\gamekee-ba-download\SKILL.md"
```

**Start a new OpenCode session** after installation for the skill to take effect.
</details>

<details>
<summary>Claude Code</summary>

**Skill Path**: `~/.claude/skills/gamekee-ba-download/SKILL.md`

> Per [Claude Code official docs](https://code.claude.com/docs/en/skills), Claude Code loads skills from `~/.claude/skills/`. If the official docs have been updated, follow the latest version.

```bash
git clone https://github.com/Elo-Mary/gamekee-ba-download-skill.git
```

Linux / macOS:

```bash
mkdir -p ~/.claude/skills/gamekee-ba-download
cp gamekee-ba-download-skill/SKILL.md ~/.claude/skills/gamekee-ba-download/SKILL.md
```

Windows (PowerShell):

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.claude\skills\gamekee-ba-download"
Copy-Item ".\gamekee-ba-download-skill\SKILL.md" "$env:USERPROFILE\.claude\skills\gamekee-ba-download\SKILL.md"
```

**Start a new Claude Code session** after installation for the skill to take effect.
</details>

<details>
<summary>Goose</summary>

**Skill Path**: `~/.agents/skills/gamekee-ba-download/SKILL.md`

> Per [Agent Skills standard](https://agentskills.io) and Goose docs, Goose loads skills from `~/.agents/skills/`. If Goose official docs have been updated, follow the latest version.

```bash
git clone https://github.com/Elo-Mary/gamekee-ba-download-skill.git
```

Linux / macOS:

```bash
mkdir -p ~/.agents/skills/gamekee-ba-download
cp gamekee-ba-download-skill/SKILL.md ~/.agents/skills/gamekee-ba-download/SKILL.md
```

Windows (PowerShell):

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.agents\skills\gamekee-ba-download"
Copy-Item ".\gamekee-ba-download-skill\SKILL.md" "$env:USERPROFILE\.agents\skills\gamekee-ba-download\SKILL.md"
```

**Start a new Goose session** after installation for the skill to take effect.
</details>

<details>
<summary>Cursor</summary>

**Skill Path**: `.cursor/skills/gamekee-ba-download/SKILL.md`

> Cursor's skills directory is typically `.cursor/skills/` within the project. Consult [Cursor official docs](https://docs.cursor.com) for the latest path. Reference steps below:

```bash
git clone https://github.com/Elo-Mary/gamekee-ba-download-skill.git
```

Linux / macOS:

```bash
mkdir -p .cursor/skills/gamekee-ba-download
cp gamekee-ba-download-skill/SKILL.md .cursor/skills/gamekee-ba-download/SKILL.md
```

Windows (PowerShell):

```powershell
New-Item -ItemType Directory -Force -Path ".cursor\skills\gamekee-ba-download"
Copy-Item ".\gamekee-ba-download-skill\SKILL.md" ".cursor\skills\gamekee-ba-download\SKILL.md"
```

Restart or reload Cursor after installation for the skill to take effect.
</details>

## Platform Support

| Platform | Status |
|---|---|
| OpenCode | Supported |
| Claude Code | Supported |
| Goose | Supported |
| Cursor | Supported |
| Codex CLI | Not yet supported (requires AGENTS.md embedding or plugin packaging; deferred to future.) |

## Usage

Trigger the skill from an OpenCode session with natural language, for example:

- "Download Blue Archive Memorial Lobby images to `D:\BA_images`"
- "Download the Official Introduction images from the gamekee BA wiki"
- "Download both Memorial Lobby and Official Introduction"

Optional parameters (defaults apply if omitted):

| Parameter | Default | Description |
|---|---|---|
| `target` | `hydt` | `hydt` = Memorial Lobby ｜ `gfjs` = Official Introduction ｜ `both` = download both |
| `output_dir` | current working directory | Root directory for downloaded images |
| `list_url` | `https://www.gamekee.com/ba/second/23941` | Character roster entry point (Implemented Students) |

The agent handles the full pipeline: scrape roster, extract images per character, download, verify, and retry. The process supports checkpoint resume and re-running to catch failed items. See [`SKILL.md`](./SKILL.md) for details.

## How It Works (Overview)

1. The browser opens the gamekee Implemented Students roster and extracts all character IDs.
2. In batches of 12, it opens each character's detail page and reads `img.src` using the selector matching the chosen `target`.
3. Extracted URLs are written to `batches/batchNN.json`.
4. PowerShell downloads each image, **determines format from the real file header after download** (the CDN performs content negotiation, so the URL extension is unreliable), and saves to the appropriate subdirectory.
5. Integrity check + unified retry for failed or unrendered items.

> gamekee's content JSON endpoint is blocked by Tencent EdgeOne WAF, so this skill uses the "browser reads `img.src` -> PowerShell downloads" route instead of a JSON API shortcut. Full constraints and troubleshooting in [`SKILL.md`](./SKILL.md).

## Current Scope

- **Game**: Blue Archive
- **Server**: JP
- **Image types**: Memorial Lobby, Official Introduction
- **Source**: gamekee.com Blue Archive wiki, Implemented Students page (~267 characters, including collab characters)

## Roadmap

- **More image types**: setting art (JP / TW), official art, expressions, etc. (requires album mode: split by `.header-container` text and grab all images in each section)
- **More servers**: Global / CN / TW
