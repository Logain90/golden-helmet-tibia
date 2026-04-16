# Golden Helmet — Research Context & Background Knowledge

This document captures all research conducted during the planning session on April 16, 2026.
A new Copilot session should read BOTH this file and IMPLEMENTATION.md before implementing.

**Prompt for new session:**
> "Read RESEARCH_CONTEXT.md and IMPLEMENTATION.md in this repo. These contain the full planning session research and implementation plan for a Tibia 7.1 OT server called Golden Helmet (World: Aldora). Use this knowledge and then help me implement Phase 0 onwards on Windows."

---

## 1. What amera.fibula.app Actually Is

The user referenced amera.fibula.app as the inspiration server. Research findings:

- The site returns HTTP 403 to crawlers — cannot be scraped directly
- The domain `fibula.app` confirms it runs the **Fibula C# game server** (github.com/Fibula-MMO/fibula-server)
- Fibula defaults to **Tibia 7.72 protocol** — NOT 7.1. amera.fibula.app is a 7.72 server, not 7.1.
- Fibula uses the **original CipSoft leaked sector map** from the 2007 breach (released on OTLand ~2017)
- This map is in **CipSoft's proprietary binary `.sec` format**, parsed by `fibula-parsing-cip` (C# library)
- The `.sec` format is **completely incompatible with OTBM** — cannot be opened in RME, cannot be used in TFS
- The "update feature" in fibula is a **Dependency Injection architecture** where each Tibia protocol version is a separate DLL. Swapping protocol versions means swapping 2 DI lines + adding the version DLL. This is NOT content-level updating — it's wire-protocol-level.
- Fibula has **no Lua scripting** — everything is C# + JSON data files
- Last commit: **December 2022** — unmaintained/abandoned

**Conclusion:** Fibula is not usable as the base for a 7.1 server. The "version update feature" concept is reproduced differently in TFS (content-level releases, not protocol swapping).

---

## 2. The Full OT Server Ecosystem (as of April 2026)

### Family A: TFS Lineage (C++ + Lua + MySQL)
The mainstream OT community approach.

| Project | Protocol | Status | Notes |
|---|---|---|---|
| `otland/forgottenserver` TFS 1.6 | Tibia 13.10+ | Active June 2024 | Modern only, not 7.x |
| `nekiro/TFS-1.5-Downgrades` branch `7.72` | 7.60–7.72 | **Archived Aug 2022** | Original 7.72 downport — read-only |
| `MillhioreBT/forgottenserver-downgrade` | 7.60–7.72 | **Active April 2026** | Best choice — maintained fork of nekiro's work |
| `opentibiabr/canary` | Tibia 15.00 | Very active | Modern, unrelated to 7.x |

**Chosen engine: `MillhioreBT/forgottenserver-downgrade`**
- Last commit: 1 month before April 2026
- Windows build: Visual Studio 2022 + vcpkg (confirmed `vc17/` folder and `vcpkg.json` present)
- Full Lua scripting system
- MySQL/MariaDB database
- 47 forks, 15 open issues, 1 open PR — community is using it

### Family B: CipSoft-Derived Servers (decompiled/leaked source)
New wave starting 2025 — uses the actual CipSoft code.

| Project | Protocol | Status | Notes |
|---|---|---|---|
| `fusion32/tibia-game` | 7.7 exact | Published Aug 2025 | Full decompilation of CipSoft 7.7 Linux binary to C. Linux only. |
| Tibia Relic | 7.4 | Live Jan 2026 | Built on fusion32's base, manually downgraded to 7.4 mechanics/map |
| ClassicOT | 7.10 | Active Oct 2025 | Also built on fusion32's decompiled code |
| `dennisvink/tibia-rust` | Custom (no XTEA, WebSocket) | Feb 2026 | Rust rewrite of fusion32's work, experimental |
| Tibiara / Revol Engine (by Ezzz) | 7.4 | Active | True binary reverse engineering (not translation) — closed source |
| TVP 7.4 / TVP 7.7 (The Violet Project) | 7.4 and 7.7 | Active Mar 2026 | Full distributions w/ map, 345 replies 40K views |

**Why not chosen:** All Linux-only or closed source. The decompiled code is from 7.7 — getting to 7.1 requires reversing ~6 versions. No Windows build. Community is active but infrastructure is complex.

### Family C: Browser/JS Implementations
| Project | Notes |
|---|---|
| `Inconcessus/Tibia74-JS-Engine` | Node.js + HTML5. Custom WebSocket protocol — NOT compatible with original Tibia client. Educational. |
| Gravak, minibia.com, Kotetso | Forks of above — some deployed as playable browser games |

**Why not chosen:** Custom protocol — incompatible with original Tibia clients.

### Legacy Engines
| Project | Notes |
|---|---|
| YurOTS | C++, original 7.x era engine, no maintained fork in 2025-2026. No public GitHub. Cited nostalgically but not usable. |
| Avesta (OTServ SVN 0.6.3) | Supports 7.4, 7.6, 7.7. Download links mostly dead (2012 era). Still referenced in OTLand threads. |
| `opentibia/server` | Archived 2020. Targets 8.7, NOT 7.x (common misconception). |

---

## 3. Tibia Protocol Version Differences

### Wire protocol differences (7.1 vs 7.4 vs 7.72)
| Version | Encryption | Skulls/Shields | Notes |
|---|---|---|---|
| 7.1 | None (no XTEA, no RSA) | No | Very early — plaintext protocol |
| 7.4 | None (no XTEA, no RSA) | Skulls added around here | |
| 7.72 | XTEA + RSA | Yes | Leaked CipSoft build — fibula default, MillhioreBT TFS default |
| 8.0+ | XTEA + RSA | Yes | |

### Content differences (7.1 vs 7.2 vs 7.4)
**7.1 → 7.2:**
- Primarily content changes (map additions, items)
- No major wire protocol restructuring documented
- 7.2 introduced Fibula island and some new items

**7.2 → 7.4:**
- Added Darama continent (Ankrahmun city, desert zones) — the biggest visual change
- New creatures: Sandstones, Larvas, Scarabs, Bloodcrabs, Serpent Spawns, Scorpions
- New items and equipment
- Minor protocol changes for extended world size

**7.4 → 7.6 (the biggest gameplay jump):**
- Magic system overhaul — most-cited breaking change in community
- New higher-tier spells
- Significant balance changes
- This is why "pre-7.6 spell formulas" is a specific thing to implement

**Important:** In the OT community "protocol" often means game-mechanical protocol, not TCP wire format. The TCP differences between 7.1, 7.2, and 7.4 are small — existing 7.4-era clients can likely connect to a "7.1 content-level" server without wire protocol changes.

**Our implementation:** Server speaks 7.72 wire protocol. Players use 7.72 client. They experience 7.1 **content** (map, monsters, spells, game rules). The version upgrade system (7.1 → 7.2 → 7.4) is content-level, not wire-protocol-level.

---

## 4. Available Maps Research

### ❌ fibula's map.zip
- Contains CipSoft `.sec` sector files from the 7.7 leak
- Parsed by `fibula-parsing-cip` (C# library) — incompatible with OTBM/RME
- Cannot be used in TFS at all

### ✅ Under Influence's 7.1 OTBM — PRIMARY MAP
- **Thread:** https://otland.net/threads/7-1-real-tibia-map-as-close-as-possible.270484/
- **Download:** Attachment `Tibia71_UI.rar` (ID: 46024, ~4.6 MB) in reply #12, May 27 2020
- **Format:** OTBM (7.4 base header — works with TFS)
- **Method:** Manually reverted 222222's 7.4 OTBM using old screenshots and Wayback Machine archived tibia world maps
- **Accuracy:** ~95%
- **What was reverted from 7.4:**
  - Rookgaard premium area removed, wolf island layout reverted
  - Carlin: mountains/houses outside city removed, level doors removed from MOLS and Demona
  - Thais: bridge lowered, Greenshore bridge removed, swamp removed, triangle tower removed
  - Venore: ladders replacing stairs, swamp bridge lowered, orc camps north removed
  - Edron: Warlock/Behemoth/Demon helmet quest access reverted, Annihilator removed
  - Darashia: Ankrahmun removed, mountains changed, Drefia removed, ghost ship removed
- **NO spawns included** — must populate spawns.xml manually

### ✅ 222222's TibiCAM 7.4 OTBM — REFERENCE MAP
- **Thread:** https://otland.net/threads/7-4-authentic-real-map-extracted-from-tibicam-files.270466/
- **Download:** Attachment `Tibia 7.4 Real Map - by 222222.zip` (ID: 45293, ~11 MB)
- **Format:** OTBM v0.6.0
- **Method:** Custom tool ($500, by Cjaker) extracted tile data from thousands of private + public TibiCAM recordings. Supplemented with GM interviews and screenshots. NOT the CipSoft leak.
- **Contains:** All NPCs with correct outfits, monster spawns at original positions, all book/text content, Isle of Solitude
- **Use for:** Cross-referencing spawn positions and NPC placements. Best authoritative source for 7.4 content details.

### ✅ Screenshot Library
- https://drive.google.com/drive/folders/1TU132ZkZipx2V5wHUW6GUgISjyE_6SQK
- Tibia 7.0–7.13 era screenshots, community collected

### ⚠️ Sizaro's CipSoft→OTBM Conversion (reference only)
- **Thread:** https://otland.net/threads/10-98-cipsoft-original-map-converted-from-7-7.252065/
- **Format:** OTBM but with **10.98 item IDs** — cannot drop into 7.x TFS directly
- **Value:** Contains movable items, trash on floor, decoration details that TibiCAM couldn't capture
- **Use as:** Visual reference for decoration placement, not as a usable map file

### ❌ 7.6 raw global map
- **Thread:** https://otland.net/threads/raw-global-tibia-map-7-6.170590/ (2012)
- Download link (MediaFire 2012) — almost certainly dead

### ✅ CipSoft 7.55 map (emerging — not yet OTBM)
- **Thread:** https://otland.net/threads/cipsoft-binary-sector-converter-tool-tibia-7-5-real-map.303803/ (Feb 2026)
- Contains a `map.bak` folder timestamped Nov 2005 (7.55 era) found inside the 7.7 leak
- Conversion tool: `github.com/danilopucci/cipsoft-binary-sector-converter`
- Still in `.sec` format — OTBM conversion pipeline being worked on by community

### Critical RME Rule
**NEVER add spawns via RME.** RME merges nearby spawns when saving, corrupting `spawns.xml`. Always edit `spawns.xml` directly in a text editor. Add new spawns to the XML file only.

---

## 5. Website Options Researched

### ZnoteAAC (chosen)
- **Repo:** `Znote/ZnoteAAC`
- **License:** MIT
- **PHP:** 7.4+ (tested on 5.6 and 7.4, works on 8.x)
- **Stars:** 153, Forks: 125
- **Engine mapping:** `$config['ServerEngine'] = 'TFS_03'` for MillhioreBT's TFS
- **Ships with:** `paypal_report.php`, `stripe_report.php` for donations
- **Why chosen:** Simpler setup than Gesior, MIT license, well-documented

### Gesior2012 (not chosen)
- **Repo:** `gesior/Gesior2012`
- **PHP:** 7.0–8.0 with MariaDB
- **Requires:** This critical `my.ini`/`my.cnf` change: `sql_mode=''` (without it, queries fail due to strict mode)
- **Requires extensions:** `bcmath curl dom gd json mbstring mysql pdo xml`
- **Why not chosen:** More complex setup, GPL license, requires MySQL strict mode disabled

---

## 6. Windows Build Stack Details

### Visual Studio 2022 requirement
The `vc17/theforgottenserver.vcxproj` file uses `<PlatformToolset>v143</PlatformToolset>`. This is the MSVC 2022 toolset. VS2019 uses v142 and will fail to load the project correctly.

### vcpkg dependency list (from repo's vcpkg.json)
```json
{
  "dependencies": [
    "boost-asio", "boost-iostreams", "boost-locale",
    "boost-lockfree", "boost-system", "boost-variant",
    "fmt", "libmariadb", "openssl", "pugixml", "zlib"
  ],
  "features": {
    "lua":    { "dependencies": ["lua"] },
    "luajit": { "dependencies": ["luajit"] },
    "http":   { "dependencies": ["boost-beast", "boost-json"] }
  }
}
```

Full install command for x64-windows:
```powershell
C:\vcpkg\vcpkg install --triplet x64-windows boost-iostreams boost-asio boost-system boost-variant boost-lockfree boost-locale boost-beast boost-json lua libmariadb pugixml openssl fmt zlib
```

### Laragon vs XAMPP
- **Laragon:** Modern, portable, tray icon, no config editing needed, includes HeidiSQL. Recommended.
- **XAMPP:** Widely documented in OTLand threads, ships Apache + PHP + MariaDB. Fine alternative if Laragon has issues.

---

## 7. Port Reference

| Port | Protocol | Purpose | Expose publicly? |
|---|---|---|---|
| 7171 | TCP | Login server + status queries | Yes |
| 7172 | TCP | Game server (active gameplay) | Yes |
| 80 | TCP | HTTP website | Yes |
| 443 | TCP | HTTPS (if SSL added later) | Yes |
| 3306 | TCP | MariaDB | **NO — localhost only** |

---

## 8. Experience Stages System — Technical Details

The stages system lives in `data/XML/stages.xml`. The `<config enabled="1" />` line activates it. When enabled, `rateExp` in `config.lua` is completely ignored — only the stage multipliers apply.

The last `<stage>` entry without a `maxlevel` attribute covers all levels above `minlevel` infinitely.

TFS reads this file at startup. To change rates, edit the file and restart the server (no recompile needed).

---

## 9. Lua Scripting System Overview

Every customizable game mechanic in TFS is a Lua script in `data/`. The engine calls the scripts via registered event hooks.

### Script interfaces:
| System | File location | Triggered by |
|---|---|---|
| Actions | `data/actions/*.lua` + `actions.xml` | Player uses item (right-click use) |
| Talkactions | `data/talkactions/*.lua` + `talkactions.xml` | Player types `!command` in chat |
| Spells | `data/spells/lib/*.lua` + `spells.xml` | Player casts spell (says spell words) |
| Creature scripts | `data/creaturescripts/*.lua` + `creaturescripts.xml` | Login, logout, death, levelup, kill |
| Movement | `data/movements/*.lua` + `movements.xml` | Player/item steps on/off a tile |
| Global events | `data/globalevents/*.lua` + `globalevents.xml` | Server startup, shutdown, timer intervals |
| NPC | `data/npc/*.lua` + `*.xml` | NPC dialogue interactions |

### Lua API object types (callable in scripts):
`Combat`, `Condition`, `Container`, `Creature`, `Group`, `Guild`, `House`, `Item`, `ItemType`, `Monster`, `MonsterType`, `Npc`, `Party`, `Player`, `Position`, `Tile`, `Town`, `Vocation`

### Example talkaction structure:
```lua
local myCommand = TalkAction("!hello")
function myCommand.onSay(player, words, param)
    player:sendTextMessage(MESSAGE_STATUS_CONSOLE_BLUE, "Hello, " .. player:getName())
    return false
end
myCommand:separator(" ")
myCommand:register()
```

---

## 10. ZnoteAAC Engine Setting Explanation

`$config['ServerEngine']` maps to how ZnoteAAC reads player data format from the DB:

| Value | TFS version |
|---|---|
| `TFS_02` | TFS 0.2 |
| `TFS_03` | TFS 0.3 / 0.4 — **use this for MillhioreBT's downgrade** |
| `TFS_10` | TFS 1.3 |
| `OTHIRE` | OTHire engine |

MillhioreBT's downgrade is based on TFS 1.3/1.5 code base but the DB schema and player data format is closest to TFS_03 for this fork's 7.72 downport. If highscores or character data appear wrong, try `TFS_10` as an alternative.

---

## 11. Game Rules — What Existed in Tibia 7.1

| Feature | In 7.1? | Notes |
|---|---|---|
| Skull system | ❌ No | Added later (around 7.4 era) |
| PvP shields | ❌ No | Post-7.1 |
| Blessings | ❌ No | Added much later |
| Bank system | ❌ No | Post-7.4 |
| Hotkeys | ❌ No | Added around 7.8–8.0 |
| Party system | Partial | Basic party existed |
| Vocations | ✅ Yes | 4 vocations: Sorcerer, Druid, Paladin, Knight |
| Premium areas | ✅ Yes | But Rookgaard premium island was different in 7.1 |
| Death penalty | ✅ Yes | 10% XP loss |
| NPC rune shops | ✅ Yes | Buy blank runes, conjure yourself |
| Tibia Client hotkey system | ❌ No | No F1–F12 macro system |

---

## 12. Key OTLand Resources

| URL | Purpose |
|---|---|
| https://otland.net/forums/maps.61/ | OTLand Maps forum — all map downloads |
| https://otland.net/forums/the-graveyard.488/ | Old server distributions archive |
| https://otland.net/forums/support.16/ | TFS support forum |
| https://tibia.fandom.com/wiki/Tibia_Updates | Authoritative per-version content list |
| https://tibia.fandom.com/wiki/Amera | Amera world history (retired official world, Oct 2002–Apr 2018, USA location, first US server) |
| https://fibula-mmo.github.io/articles/setup.html | Fibula setup guide (confirms map.zip = CipSoft .sec format) |
| https://github.com/Fibula-MMO/fibula-server | Fibula C# server (unmaintained Dec 2022) |
| https://github.com/MillhioreBT/forgottenserver-downgrade | The chosen engine |
| https://github.com/Znote/ZnoteAAC | The chosen website |

---

## 13. What "Amera" Is

Amera was an **official CipSoft game world** (not an OT server):
- Type: Open PvP
- Location: USA (first US-located server)
- Online: October 7, 2002
- Offline: **April 19, 2018** (merged into Solidera)
- Name origin: derived from "America"
- Notable player: Eternal Oblivion — held highest level and best skills in all of Tibia for extended period

In OT context, "Amera" evokes the era of pre-2018 Tibia gameplay. amera.fibula.app likely named itself after this world for the nostalgic association, not because it replicates the world technically.

---

## 14. Community Consensus on Best Engine for 7.x (April 2026)

| Goal | Recommended approach |
|---|---|
| 7.60–7.72 with original client | MillhioreBT's `forgottenserver-downgrade` |
| 7.4 faithful mechanics | fusion32's decompiled 7.7 → Tibia Relic approach (Linux, complex) |
| 7.4 browser play | `Inconcessus/Tibia74-JS-Engine` fork ecosystem |
| 7.1 specifically | **No dedicated maintained solution** — take MillhioreBT, strip to 7.1 content baseline |
| Modern (12.x+) | `opentibiabr/canary` |

**For Tibia 7.1 specifically:** No GitHub repo is tagged "tibia-71" (zero results). The community has not built a dedicated 7.1 engine. The approach of using MillhioreBT's 7.72 TFS downgrade and rolling back content to 7.1 era is the correct and only practical path outlined in community discussions.
