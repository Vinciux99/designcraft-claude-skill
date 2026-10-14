# DesignCraft skill for Claude

DesignCraft (ArtCraft suite, storytold/designcraft) is a clean-room, open-source Rust reimplementation of **Adobe InDesign**: spreads and parent (master) pages, text and graphic frames, threaded stories, paragraph/character styles, swatches, text wrap, tables, data merge. Native format `.designcraft` (zip + JSON); imports and exports IDML; exports PDF (also PDF/X-4, PDF/A-2b, tagged), PNG/JPEG, EPUB, HTML, text/RTF, XML. Status: **pre-alpha**. It **cannot open `.indd` files** — ask for an IDML exported from InDesign.

This repository packages a **skill** that teaches Claude to drive DesignCraft (part of the [ArtCraft suite](https://github.com/storytold/designcraft) by storytold) and that **knows how to create the MCP server and guide you through it**, on **Windows and macOS**. It is unofficial and not affiliated with storytold.

> Part of the [ArtCraft Claude skills hub](https://github.com/Vinciux99/artcraft-claude-skills): all 13 skills, a picker and a one-command installer.

## What it does
- Operating manual for DesignCraft: connection modes, ground rules, every MCP tool, recipes, CLI, control channel, command families and a safe workflow (work on copies, verify, save only with your OK).
- Built-in MCP setup assistant: Claude checks whether the server is connected, then walks you through installing it (automatic script or manual config), the launcher, authentication and verification.
- Troubleshooting table for the usual problems (server "disconnected", ports, macOS quarantine, Windows Store config location...).

## What you can ask Claude
- "Create an A4 flyer with a headline, three columns of body text and my photo at the bottom, then show me page 1."
- "Set up a 16-page magazine with facing pages, 3 columns and a 3 mm bleed."
- "Define paragraph styles for Title, Subtitle and Body and apply them to this story."
- "Thread these text frames together and tell me how much text is overset."
- "Mail-merge this CSV into my badge template, one record per page."
- "Open this IDML as a copy and export a PDF/X-4."
- "Add a footnote and a cross-reference to the results paragraph."
- "Export the document as a tagged PDF and as EPUB."

## Highlights
- Headless and attached (live) MCP modes, with the full tool table and the control protocol.
- Spread-space coordinate model, `rect = [x0,y0,x1,y1]`, and the UTF-8 **byte** offsets rule for text.
- Recipes for frames, stories, styles, placing images, PDF/IDML/EPUB export and data merge.
- `.dcs` scripts with `$N.path` references; CLI one-shot pipelines.
- Built-in guidance for creating and troubleshooting the MCP server on Windows and macOS.

## App-specific notes
- Kit port **7981** (upstream docs use 7979, which VectorCraft uses here); live MCP `mcp --connect 7981`; headless `mcp`.
- The control channel has no authentication and loopback only; no path sandbox.
- Pre-alpha; cannot open `.indd` (use IDML).

## Get the skill

| Where | How |
|---|---|
| **Claude app** (claude.ai / Desktop) | Download [`designcraft-skill.zip](https://github.com/Vinciux99/designcraft-claude-skill/releases/latest/download/designcraft-skill.zip)`, then Customize > Skills > "+" > Create skill > Upload a skill. Needs Settings > Capabilities > "Code execution and file creation" on. |
| **Claude Code, plugin** | `/plugin marketplace add Vinciux99/artcraft-claude-skills` then `/plugin install designcraft@artcraft` |
| **Claude Code, manual** | copy `skills/designcraft/` to `~/.claude/skills/designcraft/` (the setup script below does it) |

## Create the MCP server

Prerequisite: install DesignCraft from [upstream releases](https://github.com/storytold/designcraft/releases) (Windows `.msi`, macOS `.dmg`) and Claude Desktop and/or Claude Code.

**One command (recommended)**

Windows (PowerShell):
```powershell
powershell -NoProfile -ExecutionPolicy Bypass -Command "& ([scriptblock]::Create((irm https://raw.githubusercontent.com/Vinciux99/artcraft-claude-skills/main/install.ps1))) -Apps designcraft"
```
macOS (Terminal):
```sh
curl -fsSL https://raw.githubusercontent.com/Vinciux99/artcraft-claude-skills/main/install.sh | bash -s -- --apps designcraft
```
Add `-DryRun` (Windows) or `--dry-run` (macOS) to preview, and `-Mode live|headless|both` / `--mode ...` to choose the mode.

**Or from a clone:** double-click `setup-windows.cmd` (Windows) or `setup-macos.command` (macOS; first time right-click > Open).

**Or just ask Claude:** once the skill is installed, say "set up the DesignCraft MCP server" and Claude will guide you step by step.

After setup: fully quit and reopen Claude, open DesignCraft with the **"DesignCraft (Claude)"** launcher, then ask Claude to use it. If the server shows "disconnected", the app was not running when Claude started: open it with the launcher and re-enable the server, or use headless mode.

What the setup changes (all backed up or reversible): Claude Desktop config (`mcpServers.designcraft`, backup `<file>.backup-<timestamp>`), the launcher, Claude Code's user-scope MCP list and `~/.claude/skills/designcraft/`.

## Repository layout
```
skills/designcraft/SKILL.md      the skill
app.json                   machine-readable settings (ports, paths, MCP args) used by the scripts
scripts/                   artcraft-setup.ps1 (Windows), artcraft-setup.sh (macOS), json-tool.js
setup-windows.cmd          double-click wrapper
setup-macos.command        double-click wrapper
.claude-plugin/plugin.json Claude Code plugin manifest
.github/workflows/ci.yml   validation, setup tests on Windows and macOS, release zip
```

## CI and releases
Every push validates `app.json`, `SKILL.md` and the scripts, then runs the setup scripts against a fake install on **Windows** and **macOS** runners. Every push to `main` builds `designcraft-skill.zip` and publishes (or updates) the GitHub release `v1.0.0`.

## Status and caveats
- Windows paths and arguments come from a working install. The macOS paths (`/Applications/DesignCraft.app`, CLI location and release asset name) follow the upstream packaging but have **not** been verified on a Mac; the skill marks unverified items and the setup script reports what it could not find.
- DesignCraft is a young open-source project; behaviour can change between releases. `list_commands` / `designcraft-cli commands` is always the source of truth.
- Unofficial community packaging, MIT licensed (see `LICENSE`).
