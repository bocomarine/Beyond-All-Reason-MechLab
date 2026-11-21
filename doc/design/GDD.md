# BAR MechLab & World Grid — Game Design Document

Status: Draft v0.1 (Nov 21, 2026). Owner: Boco. Repos:
- Game (Lua): https://github.com/bocomarine/Beyond-All-Reason-MechLab
- Engine (C++): https://github.com/bocomarine/RecoilEngine (fork of beyond-all-reason/spring)

Items marked **[verify]** are design assumptions or estimates not yet measured.

---

## 1. Vision

Beyond All Reason, expanded in two directions:

1. **MechLab** — MechCommander 2–style loadout customization. Players choose weapons and equipment for a unit at the factory before it is built.
2. **World Grid** — a persistent campaign world made of a grid of 32×32 sector maps. Armies move between sectors, and battles are fought in real time inside each sector.

Pillars:
- Player-authored armies: your loadouts are your identity.
- Fast movement between sectors: never stuck behind a load screen to keep the war going.
- Built on the stock engine first. Engine changes only when measurements demand them.

---

## 2. Feature A: MechLab (build-time loadouts)

### 2.1 Player experience
- A player selects a refittable chassis in a factory's build menu and opens a loadout panel.
- They fill hardpoint slots with components while watching a heat bar and a cost readout.
- The factory builds that exact variant. **Loadouts are fixed once built.** No in-field refits (owner decision).
- Saved loadouts appear as quick-build buttons.

### 2.2 Rules (adapted from MechCommander 2)
MC2 reference: `alariq/mc2`, `code/logisticsvariant.cpp` `canAddComponent()`.
- **Heat:** the sum of component heat must be ≤ the chassis heat cap. Heat sinks raise the cap.
- **Slot space:** each component has a footprint (w×h) on the chassis slot grid, and it has to fit.
- **Jump jets:** one dedicated slot, only on chassis flagged for it.
- **Tonnage:** not used (MC2 disabled it too).
- **Cost:** chassis base cost plus the sum of component metal and energy costs. Build time scales the same way.

### 2.3 Technical design
Engine constraint (certain): a unit's weapon set is fixed when its UnitDef loads (max 32 weapons, `rts/Sim/Misc/GlobalConstants.h`). `Spring.SetUnitWeaponState` can tune stats but cannot swap the weapon def.

Approach: **pre-generated variant UnitDefs.**
- `gamedata/mechlab/components.lua` holds a data table with: id, weaponDef, heat, footprint, metal, energy, and slot type.
- `gamedata/mechlab/chassis.lua` holds: base unitDef, heat cap, slot grid, jumpJets flag, and an allowed-components list.
- `gamedata/unitdefs_post.lua` generates variant UnitDefs for the allowed combinations, named like `<chassis>_ml_<hash>`, and stores the loadout in `customparams`.
  - Guard against combinatorial explosion. Cap the combinations per chassis, or generate only curated presets first. **[verify] load time and memory at N variants.**
- The synced gadget `luarules/gadgets/mechlab_build.lua` validates the requested variant (heat, slots, ownership) and hands it to the factory build queue. It treats every client request as untrusted.
- The widget `luaui/widgets/gui_mechlab.lua` draws the loadout panel (slot grid, heat bar, cost) and sends the build request. It contains no game logic.
- Reference: `luarules/gadgets/unit_evolution.lua` already converts one unit into another via `customParams.evolution_target`.

### 2.4 Prototype (Milestone M1)
- One chassis (bot) with a laser and a cannon, choose one, plus one factory.
- Pre-generated two variants, a two-button UI, and a skirmish test against AI.
- Done when both variants build, fire their own weapons, show correct costs, and the game stays in sync in a 2-player local test.

---

## 3. Feature B: World Grid (campaign of sector maps)

### 3.1 Why sectors (not one giant map)
Engine findings (certain unless noted):
- Recoil runs every unit's simulation on every client in lockstep, so every client must keep all terrain resident. Streaming terrain chunks cannot reduce simulation memory.
- No wrap-around or spherical maps. The world is a flat plane, clamped at its edges.
- Largest shipped BAR map is 32×32 lobby units (16,384 elmos). Estimated CPU terrain memory: 32×32 ≈ 0.45 GB, 64×64 ≈ 1.8 GB, 128×128 ≈ 7 GB **[verify]**. GPU texture limits hit around 254×254.
- Therefore the world is a **grid of 32×32 sector maps**.

### 3.2 World structure
- A square grid, e.g. 8×8 sectors (equivalent to a 256×256 world).
- Each sector edge has a passable or blocked flag (mountains, sea).
- Terrain is authored as one large heightmap and sliced into sectors that share their edge rows, so heights match at the seams. Tooling to evaluate: pymapconv **[verify]**.

### 3.3 Campaign layer
- The world map (strategy view) shows sector ownership, armies, resources, and battles in progress.
- Turn-based strategy moves plus real-time battles (Dark Crusade model) for v1. Async real-time is a later option.
- The campaign server and database store sectors (owner, structures summary), armies (unit variant names including MechLab loadouts, XP, health), players, and turns.

### 3.4 Moving between sectors (owner priority: as fast as possible)
A sector change is an engine restart with a new map (`Spring.Reload` / `Spring.Restart` / `Spring.Start`), so a load screen cannot be avoided. Design around it:
1. **Exit zones:** units holding in an edge zone for N seconds leave the map and become reinforcements for the neighboring sector.
2. **World-map orders:** move armies between sectors from the strategy view with no load at all. Load in only to fight personally.
3. **Matched entry:** arriving forces spawn at the edge matching their exit, in formation.
4. **Hot-join:** hopping to a sector with a live battle joins it as reinforcements.
5. **Keep reloads short:** all sector maps ship pre-installed. **[verify] reload time at 32×32 in Phase 1.**
- Multiplayer note: players in one battle share one map. A player who hops sectors leaves that battle. The server tracks player locations.

### 3.5 Technical design
- **Battle launch:** the server writes a start script with the sector map, teams, and modoptions containing the sector ID and an army/structure payload. **[verify] the maximum modoption length; chunk it if needed.**
- **Gadget** `luarules/gadgets/campaign_sector.lua`: reads `Spring.GetModOptions()`, spawns structures and arriving armies, manages exit zones, and records departures.
- **Results:** game end goes from synced code to unsynced code, then a LuaUI widget posts the results. In single player it writes a JSON file; in multiplayer it uses a socket (`LuaSocketEnabled`) or the autohost relay.
- **Unit cap:** 32,000 total per game (engine hard limit). Inactive armies live only in the campaign database, not in-engine.

### 3.6 Where the work lives
| Component | Location |
|---|---|
| MechLab data, generation, gadget, widget | Game fork |
| Campaign gadget, results widget, world-map UI | Game fork |
| Sector map pack and slicing tool | Separate maps repo/tool |
| Campaign server, DB, launcher/autohost integration | External service |
| Larger sectors, performance work | Engine fork (only if Phase 1 demands it) |

---

## 4. Milestones

| ID | Scope | Estimate (solo) |
|---|---|---|
| P1 | Headless engine build. Blank maps at 32/64/128 with 2k/5k/10k units. Measure RAM, load and reload time, frame time (Tracy), and pathfinding memory. | ~1 week |
| M1 | MechLab prototype (§2.4) | a few days to ~1 week **[verify]** |
| M2 | MechLab full rules: heat, slots, data tables, loadout UI, saved loadouts | 2–4 weeks |
| C1 | Campaign MVP: 2×2 grid of 32×32 sectors, JSON state file, AI opponents, exit zones, unit carry-over | 3–6 weeks |
| C2 | Campaign server and multiplayer launch through the autohost | 2–4 months |
| E1 | Optional engine work: 64×64 sectors, pathfinding memory reduction | after P1 data |

---

## 5. Constraints & rules for contributors
- Keep all game-state changes in synced gadgets; widgets are UI only.
- Do not raise `MAX_UNITS` (network protocol change).
- Stay compatible with stock Recoil unless a milestone explicitly targets the engine fork.
- Free, non-commercial; no proprietary assets.

## 6. Open questions
- Maximum variant count before load time and memory suffer.
- Real sector reload time.
- Campaign turn structure: simultaneous moves or sequential turns.
- AI support for MechLab variants.
