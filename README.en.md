# gamekee-ba-download-skill

![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)
![OpenCode Skill](https://img.shields.io/badge/OpenCode-Skill-blue)

> English ｜ [中文](./README.md)

An [OpenCode](https://opencode.ai) skill: lets an AI agent batch-download character images from the [Blue Archive (BA) wiki](https://www.gamekee.com/ba/) on gamekee, following a fixed pipeline.

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

- **[OpenCode](https://opencode.ai)** installed.
- **Playwright MCP** (provides `browser_navigate` / `browser_run_code_unsafe` and other browser tools; the scraping phase depends on them). Note: the base OpenCode distribution does **not** bundle Playwright MCP. Choose one of:
  - **Recommended**: install the [oh-my-openagent](https://github.com/code-yeongyu/oh-my-openagent) plugin, which configures Playwright MCP automatically and works out of the box; or
  - Manually add the MCP server in `opencode.json`:
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
  - Both options require **Node.js 18+**; Playwright downloads browser binaries automatically on first use.
- **PowerShell** (used by the `.ps1` download script): Windows ships with PowerShell 5.1; other platforms need [PowerShell Core (pwsh)](https://learn.microsoft.com/powershell/scripting/install/installing-powershell).

## Installation

> An OpenCode skill is simply a `SKILL.md` file placed in the skills directory. Pick one of the two methods below.

### Method 1: Let the agent install it (recommended)

Paste the following to your OpenCode agent:

```
Install the OpenCode skill "gamekee-ba-download" (batch-download Blue Archive character images from gamekee):
1. Get SKILL.md from the GitHub repo https://github.com/Elo-Mary/gamekee-ba-download-skill (clone the repo or download the file directly).
2. Place SKILL.md in the OpenCode skills directory at gamekee-ba-download/SKILL.md (create the directory if needed).
   Path: Linux/macOS ~/.config/opencode/skills/, Windows %USERPROFILE%\.config\opencode\skills\
3. Confirm the file is in place when done.
```

### Method 2: git clone + manual copy

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
