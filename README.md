# wow-claude

<p align="center">
  <img src="docs/screenshot.jpg" alt="The WoW Claude chat window open in Goldshire, with a message on its way to Claude Code" width="900">
</p>

Chat with your local [Claude Code](https://claude.com/claude-code) sessions from inside **World of Warcraft: Forever** — send a task, go back to questing, get pinged in-game when the answer lands. No alt-tabbing, no `/reload` per message.

- Multiple chats, each its own persistent Claude session (like separate terminals), running in parallel
- Live progress while Claude works: action count, elapsed time, the files it's editing and commands it's running
- Replies echoed into the game chat; `/r` replies to Claude when it was the last to message you
- Claude knows your character, level, zone, talents, professions and quest log (optional), and you can shift-click items, spells and quests into a message
- Claude can draw on your world map: numbered routes, quest stops and marks, with a navigator arrow that walks you from stop to stop
- Herb and ore spawns on the world map, filtered by your gathering skill (`/wcmap ore`, `/wcmap herb`)
- An **Allow & retry** button when Claude needs a command outside your allowlist
- A status light for the bridge, automatic retries, and recovery of your chats (and map layers) if the beta client wipes addon data
- Runs on Windows, and on Linux with the game under Wine

Nothing here injects code, reads game memory, or generates input. The addon uses documented addon APIs only; the companion reads your screen and writes ordinary files.

## How it works, in one paragraph

WoW addons are sandboxed: no network, no file reads at runtime. Two doors remain. **Out:** the addon draws your message as a strip of colored 4-pixel squares in the top-left corner of the screen; the bridge screen-captures that corner four times a second and decodes it. **In:** a load-on-demand addon reads its files from disk at the moment it is loaded, so the bridge writes the reply into a pool of 200 pre-made slot addons and the game loads a fresh one from a timer. Cheap "is it ready yet" checks ride on a third trick: an empty `.wav` won't play and a valid one will. Map layers travel the same way: Claude's tools hand them to the bridge, which keeps them versioned and ships them inside the slot files. Details in [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) and [docs/MAP.md](docs/MAP.md).

## Requirements

- Windows (NTFS), or Linux with the game under Wine on an **X11** session and python3 (see [docs/INSTALL-LINUX.md](docs/INSTALL-LINUX.md))
- World of Warcraft: Forever (tested on 1.60.1.69913 and 1.60.1.69977, TOC 16001), **windowed or borderless** (exclusive fullscreen blocks screen capture)
- [Node.js](https://nodejs.org) 22.2 or newer
- [Claude Code](https://claude.com/claude-code) installed and logged in (`claude --version` works)

## Install

### Windows

Step-by-step for a fresh machine, with troubleshooting: [docs/INSTALL-WINDOWS.md](docs/INSTALL-WINDOWS.md). The short version:

```powershell
git clone https://github.com/chelinho139/wow-claude
cd wow-claude
node setup.js --project "C:\path\to\the\project\you\want\to\work\on"
```

`setup.js` finds the client (pass `--wow "<client folder>"` if it can't), copies the addon into `Interface\AddOns\WoWClaude`, writes `bridge/config.json`, and generates the slot pool and signal files (≈15,000 tiny files; that's normal — the client only discovers addon files at launch, so they have to exist up front).

Then **fully quit and relaunch WoW**, enable *WoW Claude* on the AddOns screen, and start the bridge:

```
npm start               # in the current terminal (or: bridge\start.ps1)
bridge\start-window.cmd # double-click version: opens its own window
```

It restarts itself if it ever crashes. Ctrl+C (or closing the window) stops it.

### Linux (Wine)

Details and capture troubleshooting: [docs/INSTALL-LINUX.md](docs/INSTALL-LINUX.md). The short version:

```bash
git clone https://github.com/chelinho139/wow-claude
cd wow-claude
node setup.js --project ~/path/to/the/project   # finds the client in $WINEPREFIX, ~/.wine...; or pass --wow "<client folder>"
npm start
```

The bridge captures the game window through X11 (`bridge/capture_x11.py`, no packages needed) and writes the slot files straight into the Wine prefix. Check the capture once: send any message from the game and, while the strip of colored squares is in the top-left corner, run `npm run probe` in a second terminal. It saves what the capture sees to `bridge/probe.png` and says whether it decoded the strip.

### `wow-claude`: start it from the project folder

Like `claude` itself, the bridge works in the folder you start it from. Install the command once:

```powershell
npm link          # in the wow-claude folder; makes `wow-claude` available everywhere
```

Then, from any project:

```powershell
cd C:\path\to\realms
wow-claude
```

Every chat that hasn't picked its own folder now works in `realms`, and the panel's cwd line shows it. `wow-claude --project <dir>` names the folder explicitly; `npm start` inside this repo falls back to `defaultCwd` in the config. Only one bridge can run at a time (two would fight over the screen and the slot files), so this sets the default folder rather than giving you one bridge per project.

## Use

In game: `/wow-claude` opens the window. Until the bridge has answered, a **Connect** button sits where Send would be: start the bridge, click it, and the light turns green (a message typed before that stays in the box). Then click the input box, type, Enter. The reply arrives with the whisper sound; the window's light shows the bridge state (green/yellow/red, hover for details), and **Reconnect** shows up if the bridge goes quiet.

Right-clicking a chat in the left panel opens a small menu with **Rename...** and **Folder...** (right-click again to close it); the trash can on the row deletes the chat after an OK/Cancel confirm. **Folder...** sets the folder this chat's Claude works in (same as `/wow-claude cd` below); each chat keeps its own, so you can have chats on different projects side by side.

| Command | What it does |
|---|---|
| `/wow-claude` | toggle the window; the minimize button (top right) or Esc collapses it to a small bar, click the bar to expand |
| `/ai <text>` | send from the normal chat box (`/wow-claude <text>` is the same) |
| `/r <text>` | replies to Claude when Claude was the last to message you; otherwise the normal whisper reply |
| `/wow-claude new [name]` | new chat = new Claude session. Unnamed chats take their title from your first message |
| `/wow-claude chat <n\|name>` | switch chats (or click the left panel; right-click a row for Rename and Folder, its trash can deletes it) |
| `/wow-claude cd <folder>` | folder this chat's Claude works in (**Folder...** after right-clicking the chat opens the same thing as a dialog). Relative to the bridge's folder (`/wow-claude cd realms`, `/wow-claude cd ../other`), `~` works, a full path too; `/wow-claude cd` alone goes back to the bridge's default. A chat that changes folder starts a fresh Claude session there |
| `/wow-claude reset` | wipe this chat's Claude memory, keep the transcript |
| `/wow-claude context [on\|off]` | show what Claude is told about your character and location, or turn it on/off |
| `/wow-claude rename`, `/wow-claude delete`, `/wow-claude clear` | manage the current chat |
| `/wow-claude echo full\|short\|off\|<chars>` | how much of each reply to print into the game chat (default 4000 chars) |
| `/wow-claude longchat on` | let the game chat box take 4000 characters, for long `/ai` messages |
| `/wow-claude bind <key>` | hotkey: checks for a reply while waiting, otherwise toggles the window |
| `/wow-claude cancel` | stop waiting on this chat's reply |
| `/wow-claude resend` | show the strip again if the bridge missed it |
| `/wow-claude reload` | reload the UI now (also frees the slot pool) |
| `/wow-claude mode reload` | fallback transport that costs a `/reload` per step, if pixels or slots can't work |
| `/wow-claude diag`, `/wow-claude slots` | transport diagnostics |
| `/wow-claude help` | the full list |

Click any message, or `/wow-claude copy` for the last reply, to open it in a selectable box for Ctrl+C.

### Claude knows where you are

The addon tells Claude which game and client you are on, your character (name, realm, level, race, class, faction, guild), where you are (zone, subzone, map id and the coordinates the minimap shows), your money, talents, professions with their skill, and your quest log (quest ids, marking the ones ready to turn in). A few lines, sent with the addon's hello and again whenever they change, and put into Claude's system prompt by the bridge, so you can ask "what should I be doing at my level around here?" or "write me a macro for my class" without explaining yourself first. It is only a hint: for a chat about an unrelated project it changes nothing. `/wow-claude context` shows exactly what is sent; `/wow-claude context off` stops sending it (the bridge forgets it too), and `"gameContext": false` in `bridge/config.json` turns it off for good.

Along with it, every run gets [docs/WOW-ADDON-PRIMER.md](docs/WOW-ADDON-PRIMER.md): a short reference on writing addons and macros for this client (TOC layout, sandbox rules, common frames and events, where to verify an API), so "write me an addon that..." works from any folder, not just this repo. Edit the file to suit your setup; the bridge re-reads it on every run. `"primerFile": ""` in the config drops it, and `/wow-claude context off` turns it off together with the character context.

### Link items, spells and quests

Click the input box, then **shift-click** an item in your bags, a spell in the spellbook, a quest in the log, or a link in the chat: it lands in your message the way it would in the game chat. When you send, each link becomes `[Name]` in the text and its tooltip (an item's stats, a spell's description) is attached below, so Claude sees what you see when hovering it. This works from the game chat box too (`/ai is this an upgrade? [Fine Longsword]`). Without a box focused, shift-click keeps its normal meaning.

### Map, routes and gathering nodes

Claude can draw on your world map. Ask *"route me through copper and tin around here"*, *"plan the quests I can do in Westfall"* or *"where is the nearest mining trainer?"*, and the answer arrives with:

- **Layers on the world map:** numbered pins joined by lines for routes, and pins for quest givers, objectives, turn-ins, trainers, dungeon entrances or any mark. They show on the zone map and on the continent map; hover a pin for its label, click it to navigate there.
- **A navigator:** a small frame with an arrow and the distance in yards to the current stop. It starts as soon as a route on your continent arrives, advances when you get within 12 yards, and loops on farming circuits. Drag it to move it; right-click skips a stop.
- **Herb and ore spawns:** with the optional `WoWClaude_Nodes` data addon installed, every gathering spawn point of the zone you are looking at, filtered to what your skill can gather.

For this, Claude needs game data and a way to hand marks to the bridge: any tool that appends commands to the file named in `WOWCLAUDE_MAP_FILE` (set for every run) draws on the map, and Claude can also end a reply with a small ```` ```wowmap ```` block. The bridge validates the marks, keeps them in `state.json` and ships them to the game, so they survive a UI reload and the beta's saved-data wipes. The data folder this was built with (quests, NPCs, every herb and ore spawn, recipes with trainers, dungeons, a route planner) is built locally from QuestieDB, AtlasLootClassic and the vmangos DB, which can't be redistributed here; [docs/MAP.md](docs/MAP.md) describes the command format and that setup.

| Command | What it does |
|---|---|
| `/wcmap` | list the layers, navigation and node settings |
| `/wcmap ore [on\|off]`, `/wcmap herb [on\|off]` | show or hide mining / herbalism spawns on the world map |
| `/wcmap filter all\|skill` | every spawn, or only what your skill can gather (default) |
| `/wcmap hide <layer>`, `/wcmap show <layer>` | hide or show one of Claude's layers |
| `/wcmap nav <layer> [n]`, `next`, `prev`, `stop` | drive the navigator |

To remove layers for good, ask Claude ("clear the map", "remove the mining route").

### Permissions

Claude runs headless, so it can't ask you to approve a tool. `permissionMode` in `bridge/config.json` is `acceptEdits` by default (file edits inside the project are auto-approved) and `allowedTools` lists the commands it may run. Anything else is denied, and the reply grows an **Allow WebSearch, Bash(cargo:*) & retry** button: click it, the rules are added to your config permanently, and Claude resumes where it stopped. The rule is a prefix (`Bash(rm:*)` allows any `rm`), so read the button before clicking. `bypassPermissions` gives full autonomy; you decide.

## Configuration (`bridge/config.json`)

The keys you are most likely to touch. Every key, flag and environment variable is in [docs/CONFIGURATION.md](docs/CONFIGURATION.md).

| Key | Meaning |
|---|---|
| `defaultCwd` | folder for chats that haven't been given one with `/wow-claude cd` |
| `maxParallel` | how many chats may run Claude at once (default 3) |
| `permissionMode`, `allowedTools`, `model` | passed to `claude -p` |
| `claudeArgs` | extra arguments for `claude -p`, e.g. `["--setting-sources", "project,local", "--strict-mcp-config"]` to keep your user-level hooks and MCP servers out of the game's sessions |
| `gameContext` | `false` never tells Claude about your character, whatever the addon sends (default `true`) |
| `primerFile` | the addon/macro primer appended with the context (default `docs/WOW-ADDON-PRIMER.md`; `""` = none) |
| `capture.processName` | the game exe without `.exe` (`WowB` for Forever); set by `setup.js` |
| `capture.keepComposited`, `capture.windowName` | Linux: keep the compositor drawing the game window (if the probe sees black), or find the window by title |
| `slots`, `actMax`, `presenceMax` | pool sizes; must match the constants at the top of `WoWClaude.lua` if you change them |
| `timeoutMs` | kill a run that takes longer than this (default 30 min) |

## Troubleshooting

- **Connect says "No answer from the bridge" / light stays red**: is the bridge running? Is the game window on screen and not minimized, with nothing covering its top-left corner? Exclusive fullscreen blocks capture. `bridge.log` shows `strip #N` when a message is decoded and `strip seen but rejected: ...` when one is misread.
- **Linux: `bridge.log` keeps saying `waiting for WowB window`**: the game isn't running or its window has another name: set `capture.processName` to the exe name, or `capture.windowName` to part of the window title. On Wayland the capture can't see other windows; use an X11 session.
- **Linux: `npm run probe` shows a black or stale picture**: the compositor is letting the game present on its own. Try `"keepComposited": true` under `capture`, then windowed mode, then `nvidia-settings -a AllowFlipping=0` on NVIDIA; `/wow-claude mode reload` works without any capture.
- **No herb/ore pins after `/wcmap ore`**: `/wcmap` says whether the `WoWClaude_Nodes` data addon is installed; pins show on zone maps only, and with `filter skill` only what your skill can gather.
- **Reply never appears but `bridge.log` says `done`** — `/wow-claude slots`; if the pool is empty, `/wow-claude reload` frees it and picks the reply up via the fallback path.
- **"Reply slots not installed"** — `node bridge/install-slots.js`, then restart WoW.
- **Chats vanished after a reload** — the beta client sometimes wipes addon saved data. The bridge keeps `transcripts.json` and sends your chats back automatically on the next message.
- **`/wow-claude diag` says the sound channel is unusable** — the cheap readiness checks and heartbeat are off; everything still works through slot polls, just with coarser progress. If it says a valid file reports as unplayable, WoW hasn't been restarted since the files were created.

## Documentation

- [docs/INSTALL-WINDOWS.md](docs/INSTALL-WINDOWS.md): step-by-step install on a fresh machine, with troubleshooting
- [docs/CONFIGURATION.md](docs/CONFIGURATION.md): every config key, command-line flag and environment variable
- [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md): how the pixel strip, slot pool and signal files work, and why
- [docs/INSTALL-LINUX.md](docs/INSTALL-LINUX.md): Linux + Wine install, and how to check the screen capture
- [docs/MAP.md](docs/MAP.md): map layers, the navigator and herb/ore nodes (the copilot)
- [CONTRIBUTING.md](CONTRIBUTING.md): repo layout, running the tests, conventions
- [CHANGELOG.md](CHANGELOG.md): release notes

## Development

```
npm install
npm test          # everything except the live test; runs on Windows (CI, .github/workflows/test.yml) and Linux
npm run test:live # runs the bridge in a sandbox with a real Claude call
```

Layout: `addon/WoWClaude` is the addon (`WoWClaude.lua` the chat and transport, `Map.lua` the map layers, navigator and nodes), `bridge/` the companion (`bridge.js` does I/O and processes, `protocol.js` is the pure part, `capture.ps1` / `capture_x11.py` the screen capture per platform), `docs/` the design and reference, `tests/` the checks (`map_test.js` and `map_addon_test.js` cover the map protocol and `Map.lua` in a Lua VM). After editing the addon, copy it into the game folder (`node setup.js` does that too) and `/reload`. What each test covers, and the conventions for changes, are in [CONTRIBUTING.md](CONTRIBUTING.md).

## Credits

- [0xInuarashi's wow-forever-codex](https://github.com/0xinuarashi/wow-forever-codex) measured the client's file-loading rules on a live Forever build (files must exist at launch; a not-yet-loaded file is read fresh on first use) and pioneered the pixel-out channel for Codex, with a font-metrics return channel. This project uses the same rules with load-on-demand addons instead of fonts.
- [Gethe/wow-ui-source](https://github.com/Gethe/wow-ui-source) — Blizzard's UI code, `forever` branch, used to verify every API this addon calls.
- [Questie](https://github.com/Questie/Questie) and [QuestieDB](https://github.com/Questie/QuestieDB) documented how Forever's maps work (Classic uiMapIDs, re-projected zones), which the map layer relies on.

## License

MIT — see [LICENSE](LICENSE).
