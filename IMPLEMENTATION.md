# Golden Helmet — Tibia 7.1 OT Server Implementation Guide

**World:** Aldora | **Platform:** Windows 10/11 | **Date compiled:** April 16, 2026

---

## Research Summary

This document contains everything decided and researched in the planning session. Use it to brief a new Copilot Agent session with: *"Read IMPLEMENTATION.md and implement the plan from Phase 0."*

### Why these choices were made

| Decision | Choice | Reason |
|---|---|---|
| Engine | MillhioreBT/forgottenserver-downgrade | Active (last commit April 2026), community-supported, C++ + Lua + MySQL, Windows VS2022 build |
| NOT fibula | Rejected | Unmaintained since Dec 2022, no Lua scripting, uses CipSoft .sec map format incompatible with OTBM/RME |
| NOT nekiro original | Rejected | Archived August 2022, read-only |
| amera.fibula.app | Reference only | Runs 7.72 protocol (NOT 7.1) using leaked CipSoft binary sector map — confirmed via research |
| Map | Under Influence 7.1 OTBM | Purpose-built 7.1 OTBM, ~95% accurate, manually reverted from 7.4 |
| Website | ZnoteAAC | MIT, PHP 7.4+, simpler than Gesior, shares same MySQL DB |
| Web stack | Laragon | Bundles Apache + PHP + MariaDB + HeidiSQL for Windows in one installer |
| Scripting | Lua (built into TFS) | Industry standard for OT servers — all NPCs, spells, events, commands |

---

## Map Resources (download before starting Phase 3)

### Primary Map — Tibia 7.1 OTBM
- **Thread:** https://otland.net/threads/7-1-real-tibia-map-as-close-as-possible.270484/
- **File:** `Tibia71_UI.rar` — attachment ID 46024, ~4.6 MB, posted reply #12 (May 27, 2020)
- **Format:** OTBM (7.4 base header — works with TFS)
- **What it contains:** Full Tibia 7.1 mainland — Rookgaard (no premium area), Carlin, Thais, Venore, Edron — Darashia without Ankrahmun/Drefia, no Annihilator, no triangle tower in Thais
- **Accuracy:** ~95% — author flagged tree placements south of Darashia as uncertain
- **NO spawns included** — `spawns.xml` must be populated separately

### Reference Map — TibiCAM 7.4 OTBM
- **Thread:** https://otland.net/threads/7-4-authentic-real-map-extracted-from-tibicam-files.270466/
- **File:** `Tibia 7.4 Real Map - by 222222.zip` — attachment ID 45293, ~11 MB
- **Format:** OTBM v0.6.0
- **What it contains:** Full 7.4 world reconstructed from TibiCAM recordings + GM interviews. Includes spawn positions, NPC outfits, all book text, Isle of Solitude. NOT the CipSoft leak.
- **Use for:** Cross-referencing spawn positions and NPC placements in RME

### Screenshot Archive (for uncertain tiles)
- https://drive.google.com/drive/folders/1TU132ZkZipx2V5wHUW6GUgISjyE_6SQK
- Community Google Drive — Tibia 7.0–7.13 screenshots

### Critical Map Rule
> **NEVER add spawns via RME.** RME merges nearby spawns on save, corrupting `spawns.xml`. Always edit `spawns.xml` directly in a text editor.

---

## Server Settings

| Setting | Value |
|---|---|
| Server name | `Golden Helmet` |
| World name | `Aldora` |
| IP (local dev) | `127.0.0.1` |
| Login port | `7171` |
| Game port | `7172` |
| Max players | `200` |
| Rate skill | `1` |
| Rate magic | `1` |
| Rate loot | `1` |
| Rate exp | Overridden by stages (see below) |
| Death loss | `10%` XP |
| Skull system | **Disabled** (did not exist in 7.1) |
| Blessings | **Disabled** (did not exist in 7.1) |
| World type | `pvp` |

---

## Experience Stages

File: `data/XML/stages.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<stages>
    <config enabled="1" />
    <stage minlevel="1"  maxlevel="20"  multiplier="10" />
    <stage minlevel="21" maxlevel="50"  multiplier="7"  />
    <stage minlevel="51" maxlevel="80"  multiplier="5"  />
    <stage minlevel="81" maxlevel="100" multiplier="3"  />
    <stage minlevel="101"               multiplier="2"  />
</stages>
```

Also set in `config.lua`:
```lua
stagesEnabled = true
rateExp = 1  -- ignored when stages enabled, set for clarity
```

| Level Range | Multiplier | Rationale |
|---|---|---|
| 1 – 20 | 10x | Fast hook — Rookgaard + early mainland is fun, not a grind |
| 21 – 50 | 7x | Mid-tier hunting, players learning the map |
| 51 – 80 | 5x | Serious grinding, group hunting begins |
| 81 – 100 | 3x | Endgame commitment — level 100 is a real achievement |
| 101+ | 2x | True prestige tier |

---

## Phase 0: Windows Prerequisites

Install in this exact order:

### 1. Git for Windows
- Download: https://git-scm.com/download/win
- Use all defaults during install

### 2. Visual Studio 2022 Community
- Download: https://visualstudio.microsoft.com/vs/
- During install, select workload: **"Desktop development with C++"**
- Include **English language pack** (required by the build system)
- This installs MSVC **v143 toolset** — VS2019 will NOT work (project file targets v143)

### 3. vcpkg (C++ package manager)
Run PowerShell **as Administrator**:
```powershell
cd C:\
git clone https://github.com/Microsoft/vcpkg
cd vcpkg
.\bootstrap-vcpkg.bat
.\vcpkg integrate install
```

### 4. Laragon (web stack)
- Download: https://laragon.org/
- Installs Apache + PHP + MariaDB + HeidiSQL as a single portable package
- No configuration needed — just install and "Start All"

### 5. Download and save locally (before starting)
- **Remere's Map Editor (RME)** — from OTLand downloads section (search "RME" on otland.net)
- **OTLand IP Changer** — `opentibia/loader` on GitHub — redirects Tibia client to 127.0.0.1
- **Tibia 7.72 client** — from OTLand archives (this is what players connect with)
- **Tibia 7.1 client** — from OTLand archives (content reference only)
- **Tibia71_UI.rar** — from OTLand thread 270484, attachment 46024
- **Tibia 7.4 Real Map - by 222222.zip** — from OTLand thread 270466, attachment 45293

---

## Phase 1: Game Server Build

### 1. Clone the repo
```powershell
git clone --recursive https://github.com/MillhioreBT/forgottenserver-downgrade.git
cd forgottenserver-downgrade
```

### 2. Install all C++ dependencies via vcpkg
```powershell
C:\vcpkg\vcpkg install --triplet x64-windows boost-iostreams boost-asio boost-system boost-variant boost-lockfree boost-locale boost-beast boost-json lua libmariadb pugixml openssl fmt zlib
```
This takes 10–30 minutes the first time.

### 3. Build in Visual Studio 2022
1. Open `vc17/theforgottenserver.vcxproj` (double-click — VS2022 opens automatically)
2. In the toolbar dropdowns: set **Configuration** = `Release`, **Platform** = `x64`
3. Press **F7** (Build Solution)
4. Output binary: `vc17/x64/Release/theforgottenserver-x64.exe`

### 4. Configure config.lua
Copy `config.lua.dist` → rename to `config.lua`, then set:

```lua
-- Identity
serverName = "Golden Helmet"
ip = "127.0.0.1"

-- Ports
gameProtocolPort = 7172
statusProtocolPort = 7171
loginProtocolPort = 7171

-- Database
mysqlHost = "127.0.0.1"
mysqlUser = "otserver"
mysqlPass = "REPLACE_WITH_STRONG_PASSWORD"
mysqlDatabase = "tibia_ot"
mysqlPort = 3306

-- Rates (exp overridden by stages)
stagesEnabled = true
rateExp = 1
rateSkill = 1
rateMagic = 1
rateLoot = 1

-- Game rules (classic 7.1)
deathLostPercent = 10
killsToRedSkull = 0       -- disable skull system
whiteSkullTime = 0        -- disable white skull
protectionLevel = 1
worldType = "pvp"

-- Server info
maxPlayers = 200
motd = "Welcome to Golden Helmet — World: Aldora"
ownerName = ""
ownerEmail = ""
location = "Europe"
```

### 5. Open Windows Firewall ports
Run PowerShell **as Administrator**:
```powershell
New-NetFirewallRule -DisplayName "TFS Login" -Direction Inbound -Protocol TCP -LocalPort 7171 -Action Allow
New-NetFirewallRule -DisplayName "TFS Game"  -Direction Inbound -Protocol TCP -LocalPort 7172 -Action Allow
New-NetFirewallRule -DisplayName "HTTP"      -Direction Inbound -Protocol TCP -LocalPort 80   -Action Allow
```

### 6. Start the server
Double-click `theforgottenserver-x64.exe` or run from PowerShell.

---

## Phase 2: Database Setup (MariaDB via Laragon)

### 1. Start Laragon
- Launch Laragon → click **"Start All"** → Apache and MariaDB start
- MariaDB runs on localhost:3306, default root password is empty

### 2. Open HeidiSQL (bundled in Laragon)
Connect: host = `127.0.0.1`, user = `root`, password = *(blank)*, port = `3306`

### 3. Create database and user
Run in HeidiSQL query window:
```sql
CREATE DATABASE tibia_ot CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'otserver'@'localhost' IDENTIFIED BY 'REPLACE_WITH_STRONG_PASSWORD';
GRANT ALL PRIVILEGES ON tibia_ot.* TO 'otserver'@'localhost';
FLUSH PRIVILEGES;
```

### 4. Import TFS schema
HeidiSQL: **File → Run SQL File** → navigate to `forgottenserver-downgrade/schema.sql` → Open

### 5. Create initial GM account
```sql
USE tibia_ot;
INSERT INTO accounts (name, password, type, premdays) VALUES ('1', SHA1('REPLACE_WITH_PASSWORD'), 3, 65535);
```
- `type = 3` = God account
- `premdays = 65535` = permanent premium

### 6. Security check
In Laragon → Menu → MariaDB → `my.ini` → confirm:
```ini
[mysqld]
bind-address = 127.0.0.1
```
**Never expose port 3306 publicly.**

---

## Phase 3: Tibia 7.1 Content Setup

### 3a. Map Setup

1. Extract `Tibia71_UI.rar` — get the `.otbm` file out
2. Rename it to `world.otbm` and copy to `data/world/` inside the TFS folder
3. In `config.lua`, verify: `mapName = "world"` (no extension)
4. Open `world.otbm` in RME: **File → Open**
5. Simultaneously open `Tibia 7.4 Real Map - by 222222.zip` contents in a second RME window (for spawn reference)
6. In RME: **Map → Properties** — note the header version, leave it as-is unless TFS reports an error
7. **NEVER add spawns in RME** — edit `data/world/spawns.xml` directly in a text editor

### Populating spawns.xml
Open `data/world/spawns.xml` from the 7.4 reference map. Copy spawn entries from there, removing post-7.1 monsters. 

**Remove these monster names from spawns.xml (post-7.1, not in 7.1):**
- Larva
- Scarab
- Blood Crab
- Serpent Spawn
- Sandstone
- Stone Golem (verify — may be post-7.1)
- Ancient Scarab
- Demon Skeleton (replace with Skeleton if uncertain)

**Keep all classic monsters:** Troll, Orc, Cyclops, Dragon, Demon, Bear, Wolf, Ghoul, Skeleton, Vampire, Minotaur, Giant Spider, Fire Devil, Witch, Banshee, Necromancer, Monk, Elf, Dwarf, etc.

**Screenshot library** for uncertain tile positions: https://drive.google.com/drive/folders/1TU132ZkZipx2V5wHUW6GUgISjyE_6SQK

### 3b. Items
- Source a 7.1-era `items.otb` from OTLand archives (search "7.1 items.otb")
- Alternatively use the 7.4 version for initial testing — item sets are close
- Edit `data/items/items.xml`: remove items that did not exist in 7.1 (cross-reference TibiaWiki version history)

### 3c. Spells
Edit `data/spells/spells.xml`:
- Remove spells added after 7.1
- Key rule: the 7.6 magic system overhaul changed damage formulas — use pre-7.6 values
- Cross-reference: https://tibia.fandom.com/wiki/Tibia_Updates — filter by version

### 3d. Monsters
Remove these XML files from `data/monster/`:
- `Larva.xml`, `Scarab.xml`, `Blood Crab.xml`, `Serpent Spawn.xml`, `Sandstone.xml`
- Also remove from `data/monster/monsters.xml` (the index file) entries for removed monsters

### 3e. Game rules in config.lua (already set in Phase 1 — confirm these are present)
```lua
killsToRedSkull = 0    -- no skull system in 7.1
whiteSkullTime = 0     -- no white skull
deathLostPercent = 10  -- lose 10% XP on death
```

---

## Phase 4: Lua Scripting

### data/ folder structure overview
```
data/
  spells/
    spells.xml          — spell registration (name, words, mana cost, vocation)
    lib/
      *.lua             — spell implementations (damage, effects, conditions)
  npc/
    *.xml               — NPC registration (name, position, outfit)
    *.lua               — NPC dialogue and shop logic
  monster/
    *.xml               — monster stats (HP, speed, loot table, summon cost)
    monsters.xml        — index of all monster files
  actions/
    *.lua               — item use events (rune creation, door keys, quest items)
    actions.xml         — action registration
  talkactions/
    *.lua               — !command implementations
    talkactions.xml     — talkaction registration
  creaturescripts/
    *.lua               — login/logout/death/advance hooks
    creaturescripts.xml — event registration
  movements/
    *.lua               — step-on/step-off tile events
    movements.xml       — movement registration
  globalevents/
    *.lua               — server startup, shutdown, timed interval events
    globalevents.xml    — event registration
```

### Key Lua scripts to implement for 7.1

#### 1. GM Commands (data/talkactions/)
```lua
-- !goto [player name] — teleport self to player
-- !bring [player name] — bring player to self
-- !kick [player name] — kick player from server
-- !ban [account number] — ban account
-- !item [itemid] [count] — create item in backpack
-- !reload [system] — reload scripts without restart
```

#### 2. Classic death script (data/creaturescripts/)
- Lose exactly 10% of current XP (with floor at 0 for that level)
- No bless mechanic — remove any blessing checks
- Standard respawn at temple

#### 3. NPC rune shop (data/npc/)
Sorcerer/Druid NPC sells blank runes. Example shop entry:
```lua
shopModule:addBuyableItem({"blank rune"}, 225, 226, "blank rune")
```

#### 4. Classic spell formulas (data/spells/lib/)
Pre-7.6 damage values — implement and verify against TibiaWiki 7.1 spell data:
- **HMM (Exori Vis):** old formula, lower max than post-7.6
- **Exura:** old heal formula
- **GFB (Exevo Gran Mas Flam):** original AoE flame
- **ADR (Exevo Gran Mas Vis):** original AoE energy

#### 5. No-skull enforcement (data/talkactions/ or config.lua)
Skulls are disabled via `killsToRedSkull = 0` — no additional Lua needed.

---

## Phase 5: Website — ZnoteAAC

### 1. Clone ZnoteAAC into Laragon's web root
```powershell
git clone https://github.com/Znote/ZnoteAAC.git C:\laragon\www\
```
(Or clone into a subfolder like `C:\laragon\www\golden-helmet\` and access at `http://localhost/golden-helmet/`)

### 2. First-run SQL
Open `http://localhost/` in a browser **before creating config.php**.
The page shows SQL statements to run → paste them into HeidiSQL → run against `tibia_ot` DB → refresh browser.

### 3. Create config.php
Copy `config.php.dist` → rename to `config.php`. Set:
```php
$config['ServerEngine'] = 'TFS_03';      // MillhioreBT TFS version mapping

$config['serverName'] = 'Golden Helmet';
$config['serverLocation'] = 'Europe';
$config['serverUrl'] = 'http://localhost';
$config['ip'] = '127.0.0.1';

$config['page_admin_access'] = 'YourGMCharacterName';  // exact in-game character name

// MySQL — same database as game server
$config['sql_host'] = '127.0.0.1';
$config['sql_user'] = 'otserver';
$config['sql_pass'] = 'REPLACE_WITH_STRONG_PASSWORD';
$config['sql_db']   = 'tibia_ot';
```

### 4. Fix cache directory permissions
Right-click `C:\laragon\www\engine\cache\` → **Properties → Security** → give your Windows user **Write** permission.

### 5. Feature overview
| Feature | How to access |
|---|---|
| Account registration | `http://localhost/register.php` — built-in |
| Highscores | `http://localhost/highscores.php` — auto from `players` table |
| News | Post via `http://localhost/admin.php` |
| Guilds | Auto from `guilds` table in DB |
| Character management | `http://localhost/account.php` |
| Admin panel | `http://localhost/admin.php` — ban, manage accounts |

### 6. Donation system
**Stripe:**
```php
$config['stripe']['secret_key'] = 'sk_test_XXXX';
$config['stripe']['price_per_coin'] = 0.10;  // €0.10 per premium coin
```
Webhook endpoint: `http://yourserver.com/stripe_report.php`
Credits stored in: `accounts.premium_points` column

**PayPal:**
```php
$config['paypal']['business'] = 'youremail@paypal.com';
$config['paypal']['currency']  = 'EUR';
```
IPN endpoint: `http://yourserver.com/paypal_report.php`

### 7. Custom theme
Edit files in `layout/` directory — HTML templates and CSS.
Reference amera.fibula.app for visual style inspiration (dark medieval).

---

## Phase 6: Version Update System

### Project Git repo structure
```
golden-helmet-server/
  releases/
    v7.1/
      world.otbm         — baseline 7.1 map
      spawns.xml         — 7.1 spawn table
      items.otb          — 7.1 item definitions
      data/              — baseline data folder snapshot
    v7.2/
      world.otbm         — updated map (if changed)
      spawns.xml         — updated spawns
      data/              — only changed files
    v7.4/
      world.otbm         — Darama/Ankrahmun added
      spawns.xml         — desert monster spawns added
      data/
  migrations/
    v7.2.sql             — DB schema additions for v7.2
    v7.4.sql             — DB schema additions for v7.4
  scripts/
    deploy-update.bat    — deployment script
  IMPLEMENTATION.md      — this file
```

### deploy-update.bat
```batch
@echo off
SET VERSION=%1
SET SERVER_DIR=C:\golden-helmet\server
SET RELEASES_DIR=C:\golden-helmet\releases

echo [1/4] Stopping server...
taskkill /IM theforgottenserver-x64.exe /F 2>nul

echo [2/4] Backing up current data...
xcopy /E /Y /I "%SERVER_DIR%\data" "%SERVER_DIR%\data_backup"

echo [3/4] Applying version %VERSION% files...
xcopy /E /Y /I "%RELEASES_DIR%\%VERSION%\*" "%SERVER_DIR%\"

echo [4/4] Running database migrations...
mysql -u otserver -p tibia_ot < "migrations\%VERSION%.sql"

echo Starting server...
start "" "%SERVER_DIR%\theforgottenserver-x64.exe"
echo Done. Server started.
```

Usage: `deploy-update.bat v7.2`

### Content roadmap
| Update | Map changes | Monster changes | Content |
|---|---|---|---|
| **v7.1** (launch) | Under Influence 7.1 OTBM | Classic monsters only | All classic quests, 4 vocations, no skulls |
| **v7.2** | Minor tile edits | +Wolf, Minotaur variants | New quest areas, new items per TibiaWiki 7.2 notes |
| **v7.4** | Add Darama/Ankrahmun from 222222's map | +Larva, Scarab, Bloodcrab, Serpent Spawn, Scorpion | Desert quests, new equipment tier, new spells, skull system (skulls were introduced in 7.4 era) |

**Reference for exact per-version content:** https://tibia.fandom.com/wiki/Tibia_Updates

---

## Phase 7: Verification Checklist

Run through this after completing all phases:

- [ ] `theforgottenserver-x64.exe` launches — console shows no errors, "Game server online"
- [ ] OTLand IP Changer → Tibia 7.72 client → `127.0.0.1:7171` → login screen appears
- [ ] Log in with account `1` / password → character creation or selection appears
- [ ] Enter world → character spawns on map → Aldora terrain loads correctly (7.1 Tibia)
- [ ] Walk to an NPC shop → buy a blank rune → appears in backpack
- [ ] Cast HMM (Exori Vis) on a monster → damage number appears, matches 7.1 formula
- [ ] Die → respawn in temple → confirm 10% XP lost in console/log
- [ ] No skull icon ever appears on character portrait
- [ ] Type `http://localhost/` → ZnoteAAC front page loads with "Golden Helmet" title
- [ ] Register new account on website → log in in-game → character appears in highscores
- [ ] Post a news item via `admin.php` → appears on front page
- [ ] Stripe test donation → `accounts.premium_points` incremented in HeidiSQL
- [ ] Run `deploy-update.bat v7.2` on a test copy → server restarts clean, no missing file errors

---

## Key Repos

| Repo / URL | Purpose |
|---|---|
| https://github.com/MillhioreBT/forgottenserver-downgrade | Game server C++ source |
| https://github.com/Znote/ZnoteAAC | Website PHP |
| https://github.com/Microsoft/vcpkg | C++ package manager |
| https://otland.net/threads/270484/ | Under Influence 7.1 OTBM map |
| https://otland.net/threads/270466/ | 222222's 7.4 reference OTBM map |
| https://drive.google.com/drive/folders/1TU132ZkZipx2V5wHUW6GUgISjyE_6SQK | Screenshot archive 7.0–7.13 |
| https://tibia.fandom.com/wiki/Tibia_Updates | Authoritative per-version content list |

---

## Open Question
- **Protocol display:** The server speaks the 7.72 wire protocol — players use the Tibia 7.72 client and experience 7.1 content. If you want the login screen to say "7.1" exactly (matching a 7.1 client bit-for-bit), that requires C++ changes in `src/protocol/`. Confirm if this matters before implementation.
