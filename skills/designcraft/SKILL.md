---
name: designcraft
description: Lay out pages and publications with DesignCraft (InDesign-style, ArtCraft suite) through MCP, the CLI or the control channel, and set up, connect or troubleshoot its MCP server on Windows and macOS.
---

# DesignCraft for Claude

DesignCraft (ArtCraft suite, storytold/designcraft) is a clean-room, open-source Rust reimplementation of **Adobe InDesign**: spreads and parent (master) pages, text and graphic frames, threaded stories, paragraph/character styles, swatches, text wrap, tables, data merge. Native format `.designcraft` (zip + JSON); imports and exports IDML; exports PDF (also PDF/X-4, PDF/A-2b, tagged), PNG/JPEG, EPUB, HTML, text/RTF, XML. Status: **pre-alpha**. It **cannot open `.indd` files** — ask for an IDML exported from InDesign.

**Every action is a command** with an `id` (e.g. `frame.create`) and JSON parameters. UI, CLI, MCP and control channel all call the same registry (`Session::execute(id, params)`), so anything done from a menu can be done by an agent. Every command is journaled and undoable.

## Quick check: is the MCP server connected?

1. Look at the tools you have. If tools from the MCP server `designcraft` are available (for example `list_commands`, `execute` or `inspect_*` tools) the server is connected: skip to "Operating knowledge" below and use them directly.
2. If they are missing, check whether the user installed it under the name `designcraft` or `designcraft-headless`. In Claude Desktop the server may show as **disconnected**.
3. **Most common cause of "disconnected":** the live server (`mcp --connect {port}`) exits when DesignCraft is not running at the moment Claude starts. Fix: open DesignCraft with the **"DesignCraft (Claude)" launcher** (it starts the app with its control channel on), then re-enable the server in Claude's settings or fully quit and reopen Claude.
4. If the user does not need to watch the app live, **headless mode** always starts (no window, works on files). See "Live, headless or both".
5. If nothing is configured yet, follow "Set up the MCP server" below. Ask the user which OS they use if you do not know.

## Set up the MCP server (guide the user)

Walk the user through this step by step. Ask for the OS (Windows or macOS) first. If you have a shell on the user's machine you may run the steps yourself, but **ask before editing any config file and always back it up first**. Never print tokens or keys in chat.

### 1. Prerequisites
- Install DesignCraft from its upstream releases page: https://github.com/storytold/designcraft/releases (Windows: the `.msi` installer; macOS: the `.dmg`, then drag the app to `/Applications`).
- Install Claude Desktop and/or Claude Code. For Claude Desktop, fully quit and reopen it after any config change.
- The MCP server is the **CLI** that ships with the app: `designcraft-cli`. Windows installs it next to the app (`C:\Program Files\DesignCraft\designcraft-cli.exe`, not on `PATH`). On macOS the CLI is a separate release download (asset `designcraft-cli-<version>-macos-universal.zip`) unless it is bundled in the app (unverified): the setup script below fetches it into `~/.artcraft/bin/` and clears the macOS quarantine flag.
- Upstream docs use port 7979; this kit uses **7981** so that all 13 ArtCraft apps can run together without clashing.

### 2. Option A: automatic setup (recommended)
The setup script finds the app and CLI, creates the launcher, backs up and updates Claude Desktop's config, registers the server in Claude Code (if the `claude` command exists) and installs this skill. Preview first with a dry run: nothing is changed.

Windows (PowerShell):
```powershell
powershell -NoProfile -ExecutionPolicy Bypass -Command "& ([scriptblock]::Create((irm https://raw.githubusercontent.com/Vinciux99/artcraft-claude-skills/main/install.ps1))) -Apps designcraft"
```
macOS (Terminal):
```sh
curl -fsSL https://raw.githubusercontent.com/Vinciux99/artcraft-claude-skills/main/install.sh | bash -s -- --apps designcraft
```
Or clone/download this repository and double-click `setup-windows.cmd` (Windows) or `setup-macos.command` (macOS; first time: right-click, Open). Useful options: `-Mode live|headless|both` / `--mode ...` (default `live`; `both` registers `designcraft` live and `designcraft-headless`), `-DryRun` / `--dry-run`, `-NoLauncher`, `-NoClaudeDesktop`, `-NoClaudeCode`, `-NoSkill` (macOS: `--no-launcher`, `--no-claude-desktop`, `--no-claude-code`, `--no-skill`).

What it changes: Claude Desktop config (`%APPDATA%\Claude\claude_desktop_config.json` plus the Microsoft Store/MSIX copy under `%LOCALAPPDATA%\Packages\Claude_*\LocalCache\Roaming\Claude\`; macOS `~/Library/Application Support/Claude/claude_desktop_config.json`; backup saved next to it as `<file>.backup-<timestamp>`), the launcher "DesignCraft (Claude)" (Windows: Desktop shortcut; macOS: `~/Applications/DesignCraft (Claude).app` plus a Desktop alias), Claude Code's user-scope MCP list, and `~/.claude/skills/designcraft/`.

### 3. Option B: manual setup
**Windows** - edit `%APPDATA%\Claude\claude_desktop_config.json` (if Claude was installed from the Microsoft Store, also the copy under `%LOCALAPPDATA%\Packages\Claude_*\LocalCache\Roaming\Claude\`). Keep other servers; add this inside `mcpServers`. Backslashes must be doubled in JSON.
```json
{
  "mcpServers": {
    "designcraft": {
      "command": "C:\\Program Files\\DesignCraft\\designcraft-cli.exe",
      "args": [
        "mcp",
        "--connect",
        "7981"
      ]
    },
    "designcraft-headless": {
      "command": "C:\\Program Files\\DesignCraft\\designcraft-cli.exe",
      "args": [
        "mcp"
      ]
    }
  }
}
```
**macOS** - edit `~/Library/Application Support/Claude/claude_desktop_config.json`. Use the absolute CLI path (replace `YOUR_NAME`; JSON does not expand `~`). The CLI path is unverified if your install differs: check with `ls ~/.artcraft/bin /Applications/DesignCraft.app/Contents/MacOS`.
```json
{
  "mcpServers": {
    "designcraft": {
      "command": "/Users/YOUR_NAME/.artcraft/bin/designcraft-cli",
      "args": [
        "mcp",
        "--connect",
        "7981"
      ]
    },
    "designcraft-headless": {
      "command": "/Users/YOUR_NAME/.artcraft/bin/designcraft-cli",
      "args": [
        "mcp"
      ]
    }
  }
}
```
**Launcher** (starts the app with the control channel; needed before the live server can connect):
- Windows: create a Desktop shortcut named "DesignCraft (Claude)" with target `"C:\Program Files\DesignCraft\designcraft.exe" --control 7981`
- macOS: run `open -n -a "/Applications/DesignCraft.app" --args --control 7981`, or save it as an app with Script Editor/`osacompile` (the setup script does this for you).

**Claude Code** (any OS; run once):
Windows (PowerShell):
```powershell
claude mcp add --scope user designcraft -- "C:\Program Files\DesignCraft\designcraft-cli.exe" mcp --connect 7981
claude mcp add --scope user designcraft-headless -- "C:\Program Files\DesignCraft\designcraft-cli.exe" mcp
```
macOS (Terminal):
```sh
claude mcp add --scope user designcraft -- "$HOME/.artcraft/bin/designcraft-cli" mcp --connect 7981
claude mcp add --scope user designcraft-headless -- "$HOME/.artcraft/bin/designcraft-cli" mcp
```
Check with `claude mcp list`.

### 4. Authentication
DesignCraft needs no token: the control channel listens on loopback (`127.0.0.1`) only and has no authentication. Anyone on the machine can drive the app while it is open, so close it when you are done.

### 5. Finish and verify
1. **Fully quit and reopen Claude** (Desktop: quit from the tray/menu bar, not just close the window).
2. Open DesignCraft with the **"DesignCraft (Claude)" launcher** before asking Claude to use it.
3. In Claude, confirm the server `designcraft` is connected and call its `list_commands` tool (or equivalent). A non-empty command list means it works.

### Live, headless or both
- **Live** (`designcraft`): drives the open app; the user sees every change and screenshots/UI tools work. Needs the launcher first.
- **Headless** (`designcraft-headless`): in-process engine, no window; always starts; works on files; UI/screenshot tools are not available.
- **Both**: register both and pick per task. Switch with `-Mode` / `--mode`, or by editing the config entries above.

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Server shows "disconnected" right after Claude starts | Live server exits when DesignCraft is not running | Open DesignCraft with the "DesignCraft (Claude)" launcher, then re-enable the server or restart Claude; or use headless mode |
| "Port already in use" / app opens but Claude cannot connect | Another app or an old DesignCraft instance holds port 7981 | Quit all DesignCraft windows, check the launcher uses the kit port, make sure no other ArtCraft app uses the same port |
| `command not found` / spawn error | CLI path wrong or the CLI is not installed (Windows: not on `PATH`; macOS: separate download) | Use the full absolute path in the config; macOS: run the setup script (it downloads the CLI to `~/.artcraft/bin/`) |
| macOS: "cannot be opened because the developer cannot be verified" | Gatekeeper/quarantine on the app or CLI | Right-click the app, Open; for the CLI: `xattr -dr com.apple.quarantine ~/.artcraft/bin/designcraft-cli` |
| Anyone else can drive the app | The control channel has no authentication (loopback only) | Start the app with the control channel only when needed and quit it when done |
| Config edited but nothing changed (Windows) | Claude installed from the Microsoft Store reads the MSIX copy of the config | Edit/update `%LOCALAPPDATA%\Packages\Claude_*\LocalCache\Roaming\Claude\claude_desktop_config.json` too |
| JSON config error / server not listed | Unescaped backslashes or trailing comma | Double the backslashes on Windows; validate the JSON; restore the `.backup-*` file if needed |
| App crashes on start or window is blank | Graphics backend problem | Windows: set `WGPU_BACKEND=vulkan` (or `dx12`) for one launch; avoid `gl` on some AMD drivers |
| Changes do not appear in the window | You are on the headless server | Use the live server (launcher first) |
| Quotes disappear in PowerShell JSON arguments | Windows PowerShell 5.1 quoting for `.exe` | Use `--script`/files where the CLI supports them, or single-quote the whole JSON |

## Operating knowledge

## Connecting

| Mode | How to start | What it does |
|---|---|---|
| **MCP headless** | `designcraft-cli mcp` (`--sample` opens the sample magazine) | In-process engine, no window, starts with an empty Letter document; CPU rendering |
| **MCP attached to the app** | app with `designcraft --control 7981`, then `designcraft-cli mcp --connect 7981` (or `HOST:PORT`) | Uses the app's session: edits show live in the window; enables the window tools |
| **CLI** | `designcraft-cli run / script / app / describe / commands` | One-shot jobs, pipelines, export |
| **Control channel** | `designcraft --control <port>` (or env `DESIGNCRAFT_CONTROL_PORT`) | JSON lines over TCP `127.0.0.1:<port>`; upstream docs use **7979**, this kit uses **7981** because 7979 belongs to VectorCraft (two apps on one port collide) |

- **Authentication: none.** The channel listens on `127.0.0.1` only, no token: enable it only while needed.
- **Read/write roots for automation: not documented** (no path sandbox in the docs): commands write where told, so it is on you not to overwrite originals.
- Other app options: `--sample`, `--version`. Env: `DESIGNCRAFT_CONFIG_DIR` (isolated profile), `DESIGNCRAFT_NO_PREFS`, `RUST_LOG`, `WGPU_BACKEND` (`dx12` default on Windows, `vulkan`, `gl`, `metal`).
- Logs: Windows `%APPDATA%\DesignCraft\logs\designcraft.log`; macOS `~/Library/Application Support/DesignCraft/logs/` (unverified); `designcraft.1.log` / `.2.log` = previous runs.
- The web build has no control channel.

### Choosing the channel
1. **An MCP server is available** (tools `list_commands`, `execute`, ...) → use it. Prefer `designcraft-app` when the user wants to watch the work in the window; headless `designcraft` to generate files without a GUI.
2. **A shell on the user's machine** → the CLI with its full path; for multi-step work write a `.dcs` file and run `script` (avoids JSON quoting problems in PowerShell).
3. **A cloud sandbox shell** is another machine: it cannot run the user's app or reach its `127.0.0.1:7981`. It can only prepare files (`.dcs` scripts, CSVs for data merge, images, texts).
4. **Fallback: computer use** on the GUI. Non-US keyboards: shortcuts with `[` `]` (and AltGr keys) may not arrive; prefer menus (`menu_list` / `ui.menu.invoke` when the channel exists).

## Ground rules
- **Units and coordinates:** points (1/72 in), **y points down**, in **spread** space. On a single-page spread the page's top-left is (0, 0); on a facing-pages spread the right page starts at x = page width. `inspect_document` gives page bounds per spread. 1 mm = 2.8346 pt; A4 = 595.28 × 841.89 pt; Letter = 612 × 792 pt.
- **`rect` = `[x0, y0, x1, y1]`** (corners, not width/height): on A4 with a 36 pt margin `[36, 36, 559, 200]` reaches 595 − 36.
- **Spreads and parent pages:** object-creating commands accept `spread` = index of a body spread, or `{"kind": "parent", "index": 0}` for a parent spread (index in `parents` of `inspect_document`); other values → error. Page indices of `render_page` / `export_png` are **0-based**.
- **Objects and stories** are referenced by **numeric id**: `frame.create` returns `{"id": 8, "story": 9}` (frame and its story). Threading = the ordered list of a story's frames.
- **Text offsets are UTF-8 bytes, not characters:** `anchor`/`focus` of `text.select`, `start`/`end` of `story.replaceRange` and of `find.find` results, `pos` returned by `text.placeCaret`/`text.insert`/`text.move`, `length` and `overset` of stories. `è`, `à`, `ù` are 2 bytes, `—` is 3: count in bytes. An offset in the middle of a character snaps back to its start.
- **Text:** in parameters `\n` = new paragraph, `\u2028` = forced line break.
- **Styles:** paragraph, character, object, table, cell, with based-on chains and groups. Effective attributes = defaults ← based-on ← style ← local overrides. `type.para` / `type.char` format the selection; `style.paragraph.edit` changes only the named fields. To create/apply styles find ids with `list_commands {"filter": "style"}` (not all are in the docs); in the app you can also open `ui.dialog.open {"id": "paragraphStyleOptions", "fields": {"new": true}}` (exact form: verify).
- **Parameters are NOT strict:** many commands **silently ignore unknown parameters**. A wrong name gives no error and does nothing: always check names with `describe` / `list_commands`.
- **Undo:** every command is undoable; one tool gesture (drag) is one step; a script run from the Scripts panel (`script.run`) is one step. Find the undo command id with `list_commands` ("undo"). After a drag via the channel, wait until `document.inspect` shows `canUndo` before undoing.
- **Errors in attached mode:** a failed request is **not retried** (it may have succeeded): inspect the document before retrying.
- In headless MCP, UI-only commands (`view.*`, `window.*`, `help.*`, `edit.dynamicSpelling`, `app.*` except `app.links`) and the window tools error out.
- Fonts: documents look for fonts in a `Document Fonts` folder next to the file; missing fonts show pink and are listed by Preflight (`font.list`, `font.replace`).

## MCP tools (25 + 2 resources)

| Tool | Arguments | Notes |
|---|---|---|
| `list_commands` | `filter?`, `enabledOnly?` | id, label, menu, shortcut, params, enabled |
| `execute` | `command`, `params` | Run any command |
| `batch` | `commands: [{command, params}]` or `script` (text, one `id {json}` per line) | Stops at the first error (`failedIndex`); supports `"$N.path"` |
| `inspect_document` | — | Pages, spreads, objects (id, bounds, fill, story), stories (overset), layers, styles, swatches, selection |
| `get_story` | `story` or `frame` (text frame) | Text, frames, paragraphs, lines, overset |
| `set_story_text` | `story`, `text` | Replace all text (`\n` = paragraph) |
| `new_document` | `preset`, `width`, `height`, `pages`, `facingPages`, `columns`, `gutter`, `margins`, `bleed`, `title` or `sample: true` | = `file.new` |
| `open_document` / `save_document` | path | Native `.designcraft` only |
| `place_image` | `path` or `base64`; `frame`, or the selected frame, or a new one at `x`, `y`, `width` | |
| `render_page` | `page` (0-based), `path?` | PNG returned as an image: the way to **see** the result |
| `export_png` | page + path | PNG, or JPEG if the extension is `.jpg` |
| `screenshot` | — | Whole window (attached only) |
| `select_tool` | `selection`, `type`, `rectangleFrame`, `line`, ... | |
| `pointer` | `tool?`, `events: [{kind: down/drag/up/move/doubleclick, x, y}]` | Gestures of the active tool; attached: `"space": "screen"` |
| `key` | `key` + modifiers | Headless: active tool → command shortcuts → tool shortcuts |
| `type_text` | `text` | At the insertion point or in the dialog field |
| `click` / `drag` | screen points | Real input (attached only) |
| `menu_list`, `ui_inspect`, `ui_set` | | Menu tree, UI state (`canvasRect`), change UI state (attached only) |
| `dialog_set` / `dialog_confirm` / `dialog_cancel` | | Fill and close the open dialog (attached only) |

Resources: `designcraft://document` (= `document.inspect`) and `designcraft://commands`. Tool errors: `isError: true` with a readable message. The exact argument schema is in the server's `tools/list`: if a name here disagrees, the schema wins.

## Recipes

A4 document, text frame, formatting, image, visual check (from the upstream MCP doc):
```text
new_document     {"preset": "A4", "margins": 36}
execute          {"command": "frame.create", "params": {"rect": [36, 36, 559, 200], "content": "text", "text": "Hello"}}
                 → {"id": 8, "story": 9}
set_story_text   {"story": 9, "text": "Summer Fair\nSaturday in the park"}
execute          {"command": "type.char", "params": {"size": 28}}
place_image      {"path": "/Users/me/Pictures/photo.jpg", "x": 36, "y": 220, "width": 523}
render_page      {"page": 0}
```

Like a user, with the Type tool:
```text
pointer    {"tool": "type", "events": [{"kind": "down", "x": 72, "y": 72}, {"kind": "drag", "x": 300, "y": 200}, {"kind": "up", "x": 300, "y": 200}]}
type_text  {"text": "Hello"}
key        {"key": "Enter"}
key        {"key": "Escape"}
```

Script with references to earlier results (footnote + cross-reference), from the upstream agents doc:
```text
frame.create {"rect": [72, 72, 540, 300], "content": "text", "text": "Results of the survey"}
text.select  {"story": "$0.story", "anchor": 0, "focus": 0}
footnote.insert {"text": "Survey run in 2026."}
frame.create {"rect": [72, 320, 540, 500], "content": "text", "text": "See ."}
text.select  {"story": "$3.story", "anchor": 4, "focus": 4}
xref.insert  {"paragraph": "Results", "format": "Paragraph Text & Page Number"}
```
`"$N.path"` = a field of step N's result (0-based, comments and blank lines excluded; `$last` = previous step; paths like `"$1.story"`, `"$3.ids.0"`); `"${N.path}"` interpolates inside longer strings.

More supported by the docs (exact parameters: `describe <id>`):
- **Object on a parent page:** `frame.create` with `"spread": {"kind": "parent", "index": 0}`.
- **Thread stories:** `text.thread`, `text.unthread`, `text.removeFromThread`.
- **Text in a range:** `story.replaceRange` (`start`/`end` in bytes), `story.setText`, `find.find`.
- **Case:** `type.changeCase {"case": "title"}`.
- **Drop cap:** `type.para {dropCapLines, dropCapChars, dropCapStyle, ...}`.
- **Column rules:** `object.textFrameOptions {columnRule, columnRuleWeight, columnRuleColor, ...}`.
- **Import Word/RTF with styles:** `file.place {styleMap, styleConflicts}`; PDF: `file.place {pdfPage}`, `{pdfCrop: crop|trim|bleed|art|media}`; a page of an IDML/designcraft: `file.place {layoutPage}`.
- **PDF export:** `file.exportPdf` with `{standard: "x4"}`, `{tagged}`, `{flatten: high|medium|low|ppi}`, `{media: true}`, interactive `{fullScreen, bookmarksPanel, pageLayout, view, advanceSeconds}`. The path parameter name and the value for PDF/A-2b are not documented → `describe file.exportPdf`. Alternative: CLI `--export out.pdf`.
- **Other exports:** `file.exportIdml`, `file.exportEpub {cover}`, `file.exportFixedEpub`, `file.exportHtml`, `file.exportText {format: txt|rtf|tagged}`, `file.exportXml`, `file.saveACopy`. Open IDML: `file.openIdml`.
- **Data merge:** `data.source.select {path, sheet?}` (CSV/TSV/`.xlsx`), `data.fields`, `data.placeholder.add {field, role?, story?, at?, end?, item?}` (`role`: text, image, qr, hyperlink), `data.options`, `data.preview {record}` / `data.preview.stop`, `data.merge` (creates a **new** document, the template stays intact; the old `spread` parameter is an error):
```sh
echo 'data.merge {"path": "people.csv", "records": "range", "range": "1-20"}' | designcraft-cli script - --in template.designcraft --save merged.designcraft
```

## CLI
Subcommands: `run`, `script`, `app`, `describe`, `commands`, `mcp`.
```sh
designcraft-cli commands footnote          # commands whose id/label/menu contains "footnote"
designcraft-cli describe xref.insert       # label, menu, shortcut and parameters of a command
designcraft-cli script --help              # script syntax reference
designcraft-cli run --sample --all-pages out/                                   # every page as PNG
designcraft-cli run --sample --cmd 'frame.create={"rect":[0,0,100,100]}' --export out.png
designcraft-cli run --sample --scale 0.5 --export small.png --scale 2 --export large.png
designcraft-cli script page.dcs --save page.designcraft --export page.pdf --export page.png
designcraft-cli script page.dcs --connect 7981                                  # same steps, live in the app
designcraft-cli app type.changeCase '{"case": "title"}'                          # one command to the open app
designcraft-cli app --method ui.screenshot '{"path": "/tmp/w.png"}'              # one channel method
```
(Windows: `& "C:\Program Files\DesignCraft\designcraft-cli.exe" ...`.)
- `run` processes arguments **in order**: `--page`, `--scale`, `--pdf-options` apply to the following `--export` / `--all-pages`; one with no export after it is an error. `--cmd ID=JSON` accepts `$N` references. Export format is inferred from the extension (`.pdf`, `.png`, JPEG from CLI only). `--pdf-options` syntax and whether `run` accepts `--in`/`--save` are not documented → `run --help`.
- `script FILE|-`: one `id {json}` line per command (`#` comments), or JSON lines, or a JSON array of `{command, params}`; UTF-8 with or without BOM. Documented flags: `--in`, `--save`, `--export` (repeatable), `--connect PORT`, `--keep-going` (log errors and continue).
- Output: JSON `{"completed": n, "results": [...]}`, plus `failedIndex` / `failedCommand` / `error` if a step fails (exit status ≠ 0).
- `app` talks to the open app; the default port (upstream docs show only 7979; this kit uses 7981): check `app --help`.

## Control channel
- Transport: TCP on `127.0.0.1:<port>`, **JSON lines**: one request object per line, one response per line.
- Request: object with a required string `method`, optional `id` and `params`; blank lines ignored. Anything else (not JSON, array, object without `method`, invalid UTF-8, line > **4 MiB**) → an error response and **the connection closes**: reconnect. Max **16** simultaneous connections.
- No authentication (loopback only).
- Without netcat (e.g. Windows): use `designcraft-cli app --method <method> '<json>'`.
```json
{"id": 1, "method": "engine.execute", "params": {"command": "frame.create", "params": {"rect": [36, 36, 300, 200], "content": "text"}}}
{"id": 1, "ok": true, "result": {"id": 10, "story": 11}}
```

| Method | Params | What it does |
|---|---|---|
| `engine.execute` / `ui.menu.invoke` | `{command, params}` | Run a command; `ui.menu.invoke` without `params` = choose the menu item (a "…" command opens its dialog) |
| `engine.commands` | — | All commands |
| `document.inspect` | — | Pages, spreads, objects, stories, styles, swatches, selection (`canUndo`) |
| `ui.inspect` | — | Tool, UI state, view, canvas rect, perf |
| `ui.menu.list` / `ui.tool.list` | — | Menu tree / tool groups |
| `ui.tool.select` | `{tool}` | |
| `ui.pointer` | `{events: [{kind, x, y, space?: "screen"\|"canvas", clicks?}], mods?}` | `clicks: 3` = triple click; returns selection, tool and cursor |
| `ui.key` / `ui.text` | `{key, shift?, alt?, cmd?}` / `{text}` | Synthetic keyboard |
| `ui.move` / `ui.click` / `ui.drag` | screen points, `button?: left\|right\|middle` | Real egui input |
| `ui.set` | `{brightness?, panel?, rulers?, guides?, frameEdges?, baselineGrid?, textThreads?, screenMode?, zoom?, page?, fit?}` | UI state |
| `ui.dialog.open` | `{id, fields?}` | e.g. `paragraphStyleOptions` / `characterStyleOptions` with `{name, section}` or `{new: true}` |
| `ui.dialog.set` / `.confirm` / `.cancel` | `{field, value}` | Fill and close the dialog |
| `ui.resize`, `ui.focus` | | Window |
| `ui.screenshot` | `{path?}` | Window PNG |
| `ui.render` | `{path?, page?, scale?, bleed?}` | Page render (base64 without `path`) |
| `app.open` / `app.save` / `app.export` | not documented | Files: verify |
| `app.quit` | | Quits **without** asking about unsaved changes: avoid |

Object ids are numeric (from `document.inspect` / command results); panels have stable ids (e.g. `properties`, `swatches`) used by `window.panel.*`.

## Commands by family (ids cited in the docs)
Partial list: **discover the rest with `list_commands` / `designcraft-cli commands <filter>`** and parameters with `describe`. Where written `a/b/c` they are distinct commands `family.a`, `family.b`, ....

- **file.** `new`, `open`, `openIdml`, `place`, `revert`, `saveACopy`, `exportPdf`, `exportIdml`, `exportEpub`, `exportFixedEpub`, `exportHtml`, `exportText`, `exportXml`, `importXml`, `print`, `printers`, `printBooklet`
- **frame.** `create`, `grid` · **transform.** `info`, `move {copy}`, `resize {distribute}`, `set {scaleX, scaleY, rotation, shear}`
- **text.** `select`, `placeCaret`, `insert`, `move`, `thread`, `unthread`, `removeFromThread`, `release`
- **story.** `get`, `setText`, `replaceRange`, `setDirection`, `placeAndLink/links/updateLink/unlink`
- **type.** `char`, `para`, `changeCase`, `openType`, `onPath`, `pathOptions`, `storyDirection`, `ruby`, `kenten`, `tateChuYoko`, `warichu`
- **style.** `paragraph.edit`, `group`, `exportTag`, `compositeFont.list`, `compositeFont.set`, `cell.*`, `table.*`
- **object.** `fill`, `stroke`, `color`, `gradient`, `label`, `findByLabel`, `altText`, `caption {live}`, `textFrameOptions`, `fittingOptions`, `clippingPath`, `convertShape`, `pathfinder`, `qrCode`, `attributes`, `exportOptions`, `layerOptions`, `pdfLayers`, `primaryTextFrame`, `transparencyGroup`, `swapFillStroke`, `defaultFillStroke`, effects `innerGlow`, `bevel`, `satin`, `gradientFeather`, `directionalFeather`, `globalLight`
- **path.** `addAnchor`, `appendAnchor`, `deleteAnchor`, `setAnchor`, `convertAnchor`, `close`, `reverse`, `smooth`, `split`, `erase`
- **layout.** `documentSetup`, `pageSize`, `pages.toSpread`, `spreadShuffle`, `rotateSpreadView`, `createAlternate` · **page.** `transition/transitions` · **guide.** `liquid` · **liquid.** `pageRule/object`
- **swatch.** `addToSwatches`, `addUnnamed`, `newColorGroup`, `load`, `save` (`.ase`) · **color.** `convert`, `sample`, `settings`, `loadProfile` · **ink.** `list`, `options`
- **table.** `options`, `textCell`, `placeGraphic`, `sortRows`, `moveRow`, `moveColumn`, `distributeRows`, `dropCells`
- **find.** `find`, `objects`, `changeObjects` · **font.** `list`, `replace` · **hyphenation.** `addException/removeException/list` · **spelling.** `setWords/words`
- **Notes and references:** `footnote.insert`, `endnote.insert/edit/delete/list/options`, `xref.insert`, `note.new/edit/delete/convertToText/list`, `anchored.insert/options`, `hyperlink.edit`, `hyperlink.goToSource`, `bookmark.rename`, `list.define`, `math.insert`
- **Editorial work:** `changes.track/list/acceptAll/rejectAll`, `changes.accept`, `changes.reject`, `condition.new/options/delete/apply/list`, `article.new/add/remove/options/delete/list`
- **data.** `source.select`, `source.update`, `source.remove`, `fields`, `placeholder.add`, `placeholder.remove`, `options`, `preview`, `preview.stop`, `merge`
- **Books and libraries:** `book.new/open/add/remove/list/styleSource/paginate/syncStyles/exportPdf` (`.dcbook`), `library.new/open/add/place/remove/list/json/close` (`.dclib`), `links.copyTo`, `links.relinkFolder`, `place.styles`
- **Interactive:** `button.set/clear/list`, `form.set/clear/list`, `states.create/show/rename/release/list`, `media.options/get`
- **xml.** `newTag/deleteTag/tag/mapStyle/structure`, `tagText`, `loadDtd`, `deleteDtd`, `validate`
- **Preferences / scripts:** `document.preferences` (units, increments, grids; undoable, saved in the document), `prefs.set` (app), `script.run`
- **UI-only (attached app only):** `view.*` (`separations`, `flattenerPreview`, `snapPreferences`, `rotateSpread`), `window.*` (`panel.move/float/dock/activate/close/layout`, `newWorkspace`, `uiScale`, `setShortcut`, ...), `help.*` (`about`, `discord`, `github`, ...), `app.*` (`links`, `language`, `tablePanel`)

## Recommended workflow
1. **Orient:** choose the channel (above). `inspect_document` (or `document.inspect`) for pages, spreads, object/story ids, styles and swatches; in the app also `screenshot`.
2. **Open or create:** `open_document` / `file.openIdml` working on a **copy**; or `new_document` with size, margins, columns, bleed. Never open and re-save the client's original.
3. **Find commands:** `list_commands {"filter": "..."}` or `designcraft-cli commands ...`, then `describe <id>` for the exact parameter names (wrong ones are silently ignored).
4. **Edit** with `execute` or, for linked steps, `batch` / a `.dcs` script with `$N.path`. Text offsets in UTF-8 bytes; coordinates in points in spread space.
5. **Verify:** `render_page` (or `ui.render`, `run --all-pages`) and look at the PNG; `get_story` for overset (text that does not fit its frames); `inspect_document` for bounds.
6. **Save/export only when the user agrees**, under a new name (`save_document` → `.designcraft`; PDF/IDML via `file.export*` or CLI `--export`). Do not overwrite originals and do not use `app.quit` (it does not ask to save). Close the control port when done.
