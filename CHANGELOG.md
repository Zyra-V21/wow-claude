# Changelog

All notable changes to this project are recorded here. The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and the project uses [Semantic Versioning](https://semver.org/).

## [Unreleased]

### Added

- Linux support: the game under Wine on an X11 session. `bridge/capture_x11.py` (python3 + libX11 through ctypes, no packages) does what `capture.ps1` does, finding the game window by its Wine WM_CLASS (`wowb.exe`) and tolerating a few pixels of misalignment. `npm run probe` saves what the capture sees to `bridge/probe.png`. New `capture` keys: `python`, `windowName`, `keepComposited`. `setup.js` looks for the client inside Wine prefixes. See [docs/INSTALL-LINUX.md](docs/INSTALL-LINUX.md).
- Map layers drawn by Claude: numbered route pins joined by lines, quest stops and marks on the world map (zone and continent views), plus a navigator with an arrow and the distance in yards to the next stop that advances as you arrive. Claude's tools append commands to the file in `WOWCLAUDE_MAP_FILE` (set per run), or Claude ends its reply with a ```` ```wowmap ```` block. The bridge validates them, keeps the layers versioned in `state.json` and ships the whole set in the slot files, so nothing is applied twice and a client that lost its saved data gets them back on its next hello. See [docs/MAP.md](docs/MAP.md).
- Herb and ore spawns on the world map from an optional `WoWClaude_Nodes` data addon, filtered by your gathering skill. `/wcmap` toggles them (`ore`, `herb`, `filter all|skill`) and controls layers and navigation (`show`, `hide`, `nav`, `next`, `prev`, `stop`).
- The game context includes the quest log (quest ids, `*` when ready to turn in); its cap goes from 700 to 900 bytes.
- `claudeArgs` in `config.json`: extra arguments for every `claude -p` run.

- Claude is told which game and client you are on, your character (name, realm, level, race, class, faction, guild), zone and map coordinates, money, talents and professions. The addon sends these few lines with its hello and again when they change (a `c` flag and an extra field in the strip record), the bridge keeps the latest in `state.json` and passes it to every run with `--append-system-prompt`. `/wow-claude context` shows it, `/wow-claude context off` stops it (and clears the bridge's copy), `"gameContext": false` in `config.json` disables it on the bridge side.
- `docs/WOW-ADDON-PRIMER.md`, a short reference on writing addons and macros for the Forever client, goes into the system prompt with the game context on every run, whatever folder the chat works in. `primerFile` in `config.json` points elsewhere or (`""`) drops it; edits are picked up without a restart.
- Shift-click an item, spell, quest or name while the addon's input box has focus to link it into the message, as in the game chat (hooked on `ChatFrameUtil.InsertLink`, the modern chat code the Forever client runs; the old `ChatEdit_InsertLink` global is used only where that is missing). On send, each link becomes `[Name]` in the text and its tooltip is appended in a "Linked from the game" block, so Claude can read an item's stats or a spell's description. Links typed in the game chat (`/ai … [item]`) get the same treatment.

### Changed

- Rename and Folder moved off the bottom row into a small menu that opens when you right-click a chat in the left panel.
- Each chat row has a trash can that deletes the chat after an OK/Cancel confirm; `/wow-claude delete` still deletes without asking.
- Send sits at the right end of the input box instead of at the left of the bottom row.
- The bridge no longer exits when the addon folder is missing from `Interface\AddOns`; it logs one warning and the banner shows `addon : NOT INSTALLED`.

### Fixed

- Professions in the game context were always empty on Forever: the client only has `C_SkillInfo` (one table per skill line), not the classic `GetNumSkillLines`/`GetSkillLineInfo` globals, which stay as fallback. Child lines that repeat their parent are skipped.
- Deleting a chat in game now tells the bridge to forget its transcript and Claude session (a `d` strip record), so a later restore no longer brings the chat back. Deletions made while the bridge was away are resent with the next hello.
- On clients where the sound-file self-test fails (an empty `.wav` reports as playable), the addon can't hear the bridge's 30-second presence beats, and the status light went yellow 90 s after every reply, so each new message needed a Reconnect click and burned a slot. In that mode the light now allows for the 10-minute idle slot poll (green up to 12 min without news, "down" after 22), so it stays green while the bridge is running.
- A message sent while the light is not green is now sent automatically once the bridge answers the reconnect, instead of waiting for a second click on Send.

## [0.3.0] - 2026-09-22

First public release.

### Added

- In-game chat window (`/wow-claude`) with multiple chats, each backed by its own persistent Claude Code session, running in parallel up to `maxParallel`.
- Outbound transport: messages drawn as a pixel strip in the top-left corner and decoded by a PowerShell screen capture.
- Inbound transport: a pool of 200 load-on-demand slot addons the bridge writes replies into, plus `Inbox.lua` for the `/reload` fallback.
- Empty-wav signal files for acknowledgements, reply readiness, per-action heartbeats, and a 30-second presence beat that drives the status light.
- Live progress in the working bubble: action count, elapsed time, and the files and commands Claude is touching.
- Replies echoed into the game chat; `/r` replies to Claude when it was the last to message you; `/ai <text>` sends from the chat box.
- **Allow & retry** button when Claude is denied a tool, which appends the rule to `allowedTools` and resumes.
- Per-chat working folder (`/wow-claude cd`, **Folder** button) resolved against the bridge's default folder.
- Bridge-side transcripts and automatic restore of chats after the client wipes addon saved data.
- `wow-claude` command (`npm link`) that uses the folder it is started from as the default project.
- `setup.js` installer: finds the client, copies the addon, writes `config.json`, builds the slot pool.
- Test suite: addon in a Lua VM with a stub client, protocol unit tests, slot-file round trip, codec-to-decoder round trip, and a live inject test.

[Unreleased]: https://github.com/chelinho139/wow-claude/compare/v0.3.0...HEAD
[0.3.0]: https://github.com/chelinho139/wow-claude/releases/tag/v0.3.0
