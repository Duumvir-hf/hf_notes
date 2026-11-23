# Enemy and campaign ship pools: how SG, TG, garrisons, Tarkhans and mercs are rolled (1.163)

Program: `Highfleet.exe.unpacked.exe` v1.163, image base `0x140000000`.
Re-audited 2026-11-12 from the live Ghidra program (decompiles + byte-level
call-site disassembly + property serializers). Not validated in game. Tags: [code].

## Short answer

There are no external pool files: every ship spawn pool in HighFleet 1.163 is a
**hard-coded design-name list in `.text`**, resolved at spawn time through the
global design library `LIB_SHIPS` (`0x143a13a70`, loaded at startup from
`Objects/Designs/*.seria` by `PreLoadLibraries_` `0x140233710`), with each
design's price/strength scalar read from `design+0x15f8` (`retCreature154c_`
`0x140219980`).

An earlier pass misidentified several pools because Ghidra's decompiler dropped
`XMM3` and stack arguments on `somethingSpawnShips` and labelled `Location`
(city) pointers as `Creature`. The verified campaign and non-campaign pool map:

| Gameplay category | Function / site | Escadra `role` (`+0x3a8`) | How it actually works |
|---|---|---|---|
| **Strike Groups (SG)** | `Endgame_BuildEscadra` `0x1402077c0` (+ endgame phase-3 rider `0x1402629dd`) | `5` (`6` for Khiva wave) | Flagship vector + fill vector + `Tarantul ARM` / `Wasp`\|`Longbow` / `Typhon` extras. |
| **City Garrisons** | `somethingSpawnShips` `0x140203b80`, **pool `0`** (via `Prologue_SpawnEventEscadra` `0x1402038e0`) | `2` | 13-ship base pool rolled to `citysize / (20\|30)` budget at all 4 city-generation sites in `restOfEventsPrologueAntIsaiah` `0x14028de70`. |
| **Tactical Groups (Air & Missile)** | Post-roll `chanceA` / `chanceB` branches at tail of `somethingSpawnShips` (`0x1402058fe..0x140205adc`) | `2` docked; `3` (missile) / `4` (air) when detached as `TASKGROUP` (`0x14020e180`) | **Not a separate pool id!** Appended onto Pool-0 city fleets: `chanceA` adds `Longbow` (air TG); `chanceB` adds `Tarantul` + `Hound` + 50% `Eagle` + 50% `Tarantul MRL` (missile TG). Map UI & Intel Towers identify TGs by carrier/strategic-missile capability on non-SG fleets. |
| **Tarkhan personal ships** | `restOfEventsPrologueAntIsaiah` `0x14028f620` (campaign) & `endgameTarkhanShipNames?` `0x140238e70` (Khiva finale) | `NPC.m_ship_first` (`+0xd0`) / Khiva fleet | 11-ship `RandInt(1,11)` switch for named Tarkhans in cities (`FOR_TARKHAN`); 4-group flagship table for logic-10500 `ENDWAR_CARD` at Khiva. |
| **Mercenary / hireable city ships** | `somethingSpawnShips` **pool `2`** (`0x1402a8342`) & **pool `4`** (`0x140262bda`) | `NPC` cards (`m_tarkhan == 0`) | Pool 2 stocks type-6 mercenary cities with up to 5 paid `FOR_HIRE` pier ships; pool 4 stocks non-Khiva endgame cities with up to 3 free `FOR_JOIN` ships (cheapest of roll). |
| **Tarkhan call-in reinforcements** | `addReinforceingShips` `0x14021b8c0` | Appended to player fleet / Khiva fleet | Tier 1 (8 ships) for `ASK_REINFORCE` / `DIPDIALOG_WIN`; tier 0 (5 ships) for Khiva finale escort. Full details: [mercenary-ships.md](mercenary-ships.md). |
| **Trade Convoys** | `spawnTradeConvoy?` `0x140208660` | `1` | Dedicated hardcoded 1-trader (`Pegasus`/`Odissey`/`Kastor`/`Pollux`/`Kadesh Mk1`) + fuel-range escort drafter (not CV-budgeted). |
| *(Dead / non-campaign)* **Skirmish waves** | `Fleet_GenerateShipNames` `0x140205ae0` → `Fleet_CreateRandomEscadraForSlot` `0x1401c2ba0` | `2` | 44-push (43 unique) 5-bucket catalogue used **only** in leftover `SCR_COMBAT_SKIRMISH` (`SavesSkirmish/`), never in the campaign. |
| *(Editor only)* **Shipworks TEST** | `Fleet_BuildShopOffer` `0x140206950` | `2` | Only caller is `EDC_TEST_FIGHT` (`0x14013df2e`, Shipworks `TEST` button checkboxes `SMALL`/`MID`/`LARGE`/`CARRIER`). |

---

## 1. `somethingSpawnShips` (`0x140203b80`–`0x140205adc`)

### 1.1 Real ABI (register/stack verified)

```c
void somethingSpawnShips(
    CVec<EscadraName> *out,      // RCX: output vector of 0x20-byte ship names
    float              budget,   // XMM1: target sum of design+0x15f8 (<= 0 skips roll)
    int                poolId,   // R8D: 0..4 (1 and 3 have no callers in 1.163)
    float              chanceA,  // XMM3: probability of appending "Longbow" (XMM2 unused!)
    float              chanceB,  // [RSP+0x28]: probability of appending missile TG package
    char               anySize   // [RSP+0x30]: 0 = prefer >=3 then >=2 ships; 1 = best any
);
```

### 1.2 Roll loop (`0x1402056f1`–`0x1402058f7`)

Builds a temporary `CVec<(EscadraName, float price)>` (stride `0x28`) for the
chosen `poolId`, then runs up to **10 attempts**:

* Uniform random index `HGE->RandInt(0, pool.len - 1)` (no shuffle; duplicate
  pushes double a ship's weight).
* Rejects a draw if `total + price > budget * 1.2`; aborts the attempt after 10
  consecutive rejections; stops once `total >= budget`.
* Tracks the highest-total attempt overall (`best_any`), with `>= 2` ships
  (`best_ge2`), and with `>= 3` ships (`best_ge3`); exits early if a `>= 3`
  list lands within `(0.9 * budget, 1.2 * budget)`.
* When `anySize == 0` (all campaign garrisons and type-6 merc cities), picks
  `best_ge3`, else `best_ge2`, else `best_any`. When `anySize != 0` (endgame
  pool 4 only), picks `best_any` so a single expensive ship can win.

### 1.3 Pool tables (exact push order)

* **Pool 0 — Campaign City Garrisons (13 names):**
  `Courageous, Slogger, Navarin, Ballistic, Sarma, Nimrod, Flower, Intrepid,
  Paladin Mk2, Gladiator, Gladiator MRL, Fearsome, Archangel`
* **Pool 1 — Dead (`0` callers in 1.163, 4 names):**
  `Kormoran, Negev, Nomad, Nomad Mk2`
* **Pool 2 — Mercenary-City `FOR_HIRE` Stock (28 pushes / 27 unique, `Fearsome` twice):**
  `Yars, Yars Mk2, Mockingbird, Skylark, Tanker, Rook, Lightning, Fenek,
  Navarin, Scarab, Meteor Mk3, Fatman, Flower, Intrepid, Fearsome, Jaguar,
  Paladin Mk2, Gladiator, Fearsome, Gepard, Archangel, Kormoran, Negev,
  Voskhod, Wasp, Longbow, Egida Mk2, Triumphant`
* **Pool 3 — Dead (`0` callers in 1.163, 23 names):**
  `Mockingbird, Locust, Skylark, Rook, Yars, Meteor Mk3, Lightning,
  Lightning Mk3, Fenek, Fenek Mk2, Fatman, Flower, Navarin, Scarab, Intrepid,
  Paladin Mk2, Gladiator, Fearsome, Gepard, Voskhod, Voskhod X2, Wasp, Wanderer`
* **Pool 4 — Endgame City `FOR_JOIN` Stock (14 names):**
  `Tanker, Fenek, Scarab, Fatman, Intrepid, Paladin Mk2, Gladiator, Fearsome,
  Gepard, Archangel, Kormoran, Negev, Voskhod, Wasp`

### 1.4 Post-roll Tactical Group packages (`0x1402058fe`–`0x140205adc`)

Executed after the budget roll (overshooting `budget`):

1. **Aircraft Tactical package (`chanceA`, `0x140205925`):**
   If `RandFloat(0, 1) < chanceA`, appends **`"Longbow"`**.
2. **Missile Tactical package (`chanceB`, `0x14020596d`):**
   If `RandFloat(0, 1) < chanceB`:
   * always appends **`"Tarantul"`** and **`"Hound"`**;
   * if `0.5 < RandFloat(0, 1)`, appends **`"Eagle"`** (an A-100 missile cruiser in vanilla);
   * if `0.5 < RandFloat(0, 1)`, appends **`"Tarantul MRL"`**.

### 1.5 Every live caller of `somethingSpawnShips`

| Caller | Site | `poolId` | `budget` | `chanceA` / `chanceB` | `anySize` | Result |
|---|---|---|---|---|---|---|
| `Prologue_SpawnEventEscadra` `0x1402038e0` ← 4 sites in `restOfEventsPrologueAntIsaiah` (`0x14028e9f6` Khiva, `0x14028ecc8` 6 HQ ring cities, `0x14028efe3` main city grid, `0x14028f477` prologue city grid) | `0x140203a62` | `0` | `citysize / (20\|30)` (`0` within `5000` of `"UR"` on normal) | Scaled by latitude & difficulty (`0.15` normal / `0.20` hard; `0` near `"UR"` on normal) | `0` | Builds an `Escadra` (`role = 2`) linked to the city. Fleets that hit `chanceA` or `chanceB` become the map's Air / Missile Tactical Groups! |
| `Prologue_SpawnEventGroups` `0x1402a7f60` (actually the campaign **city-type assigner**, `Location+0x25c = 2..6`, unique within `2000` units) | `0x1402a8342` | `2` | `citysize / 20.0` | `0.0` / `0.0` | `0` | **Mercenary cities (`+0x25c == 6`):** creates up to **5** `NPC` records (`m_location = city`, `m_ship_first = design`, `m_tarkhan = 0`) for the city pier `FOR_HIRE` strip. |
| `spawnEndgameShips?` `0x140261a20` | `0x140262bda` | `4` | `citysize / (20\|50)` | `0.0` / `0.0` | `1` | Non-Khiva cities in Nuclear War: clears old city NPCs, sorts rolled ships **ascending** by `design+0x15f8` (`List_MergeSortByNodeKey` `0x14005aa50`), and creates up to **3** free `FOR_JOIN` `NPC` records. |

### 1.6 How Tactical Groups work at runtime

* **While docked (`role == 2`):** Intel-tower target lookup (`Battle_ChooseDeployShip`
  `0x1401bdc20`, `TERMINAL_INTEL_BUY_TAC` branch at `0x140133ee6`) and map labels
  (`showCertainLables_` `0x14010a4e0`, `"ESCADRA_DESC_CARRIER"` / `"ESCADRA_DESC_MISSILE"`)
  classify any non-Strike-Group enemy fleet (`role != 5 && role != 6`) with
  aircraft (`+0x4e4 != 0` / `+0x490 != 0`) or strategic missiles (`+0x4ec != 0` /
  `+0x48c != 0`) as a Tactical Group.
* **Detached `TASKGROUP` (`0x14020e180`, called from `Escadra_UpdateSearchAndPursuit` `0x1400ef672`):**
  Splits fast ships off into a new `Escadra` named `"TASKGROUP"` and assigns
  `role = 3 + (sum(design+0x17cc) != 0)` — i.e., **`role = 3` (Missile Taskgroup)**
  or **`role = 4` (Air Taskgroup)**.

---

## 2. `Endgame_BuildEscadra` (Strike Groups) `0x1402077c0`–`0x14020865f`

Builds `Escadra`s with **`role = 5`**.

* **Vector A — Flagship candidates (first ship):**
  * Normal (`Profile+0x165 == 0`): `[Borey]`
  * Hard (`Profile+0x165 != 0`): `[Nomad Mk2, Nomad, Gryphon]`
  * Picks the **first entry in insertion order** with `price <= budget * 0.5`;
    if none fits, appends `A.back()`.
* **Vector B — Budget fill pool (10 consecutive-reject loop up to `1.2 * budget`):**
  * Normal: `[Kormoran, Negev, Borey]`
  * Hard: `[Gryphon, Kormoran, Negev, Borey]`
* **Post-roll extras:**
  * `Tarantul ARM`: 25% normal / 50% hard, gated on `hardmode || dist("UR", pos) >= 5000.0` (`0x14020f9a0` returns Euclidean `SQRTSS` distance at `0x14020fb47`).
  * `Wasp` or `Longbow` (50/50 `RandInt(1,2)`): 25% normal / 50% hard (ungated).
  * `Typhon`: appended unconditionally when `param_5 != 0`.
* **Call sites:**
  1. `restOfEventsPrologueAntIsaiah` `0x14028fce9`: 6 campaign-start Strike Groups
     (`role = 5`, budgets `150..350`, `typhon = 0`).
  2. `spawnEndgameShips?` phase 1 (`0x140262481`): up to 4 Khiva-quadrant assault
     groups (budget `100`, `typhon = 1`, `role` overwritten to **`6`**).
  3. `spawnEndgameShips?` phase 2 (`0x1402628ce`): up to 3 outer Strike Groups
     (`role = 5`, budget `200` normal / `300` hard, `typhon = 0`), followed by
     phase 3 (`0x1402629dd`) which directly instantiates and appends one
     **`Tarantul` (66%)** or **`Tarantul ARM` (34%)** via `0x140209b00`.

---

## 3. Tarkhan personal ships (`0x14028f620` & `0x140238e70`)

See [mercenary-ships.md](mercenary-ships.md) for the full `NPC` layout (`m_tarkhan`
at `+0xf9`, `m_location` at `+0xb0`, `m_ship_first` at `+0xd0`).

1. **Campaign Tarkhans (`restOfEventsPrologueAntIsaiah` `0x14028f570`–`0x14028f788`):**
   Places unjoined Tarkhan NPCs (`m_tarkhan != 0`) into cities at least `1500`
   units apart in Y and rolls `m_ship_first` with `RandInt(1, 11)`:
   `Intrepid Mk2, Jaguar Mk2, Paladin Mk2, Gladiator, Gepard, Gepard Mk2,
   Archangel, Wasp, Longbow, Kormoran, Negev`.
   *(At profile init `saveEventsToProfile?` `0x1402a6926` / `0x1402a6ead`,
   `HARISHID` gets `Gladiator` and `GOVERNOR` gets `Varyag`; all others default
   to `Paladin Mk2` before the prologue loop assigns city Tarkhans.)*
2. **Endgame Khiva Tarkhan Flagships (`endgameTarkhanShipNames?` `0x140238e70`):**
   Called from `creatureLogic` panel id `10500` (`0x1400430df`, `ENDWAR_CARD`),
   which picks one ship uniformly by `obj+0x3e4` group (`1..4`) and calls
   `addReinforceingShips(khivaFleet, name, tier=0, ...)`:
   * **Group 1 (14):** `Skylark, Tanker, Fatman, Intrepid Mk2, Intrepid Mk3,
     Wanderer Mk2, Jaguar Mk2, Paladin Mk2, Gladiator, Fearsome, Gepard Mk2,
     Slingshot, Wasp, Voskhod`
   * **Group 2 (6):** `Archangel, Egida Mk2, Triumphant, Longbow, Kormoran, Negev`
   * **Group 3 (4):** `Triumphant, Longbow, Kormoran, Negev`
   * **Group 4 (3):** `Vega, Nomad, Nomad Mk2`

---

## 4. Trade Convoys (`spawnTradeConvoy?` `0x140208660`, `role = 1`)

Dedicated non-budget composer (up to 30 convoys at campaign start via
`0x14028fec0`, plus quest convoy `CROW50` at `0x1400a9617`):

* **1 trade ship** (`RandInt(1,4)`): `Pegasus` (12.5%), `Odissey` (12.5%),
  `Kastor` (25%), `Pollux` (25%), `Kadesh Mk1` (25%).
* **Escorts** (cap 2 normal / 4 hard, gated by fuel range `(total_fuel / total_rate) * min_fuel > 3e6`, not CV):
  9 standard hulls (`Intrepid, Scarab, Ballistic, Talos, Slogger, Lightning,
  Meteor Mk3, Meteor Mk2, Courageous`), plus on hardmode when trader roll `> 1`
  and `rand > 0.75` one `Kormoran` or `Negev`.

---

## 5. Non-campaign pools (leftover Skirmish & Shipworks Editor)

* **`Fleet_GenerateShipNames` (`0x140205ae0`) → `Fleet_CreateRandomEscadra` (`0x1401a7120`) → `Fleet_CreateRandomEscadraForSlot` (`0x1401c2ba0`):**
  44 pushes (43 unique, `Fearsome` twice) bucketed by price (`<=15`, `<=25`,
  `<=40`, `<=75`, `>75`). **Never called in the campaign:** its only two callers
  are `Scene_ResetGlobalLinks` (`0x14028dd3f`, gated on `0x147fc6f80` which
  `SetSkirmishFlagsFromLevelName` `0x1402db350` sets only for `*Skirmish.seria`)
  and `SomethingWithLevelSerias` (`0x1402c3fa2`, inside `scene == "SCR_COMBAT_SKIRMISH"`).
* **`Fleet_BuildShopOffer` (`0x140206950`):**
  Despite its Ghidra name, its sole caller is `!ClickFunc` `0x14013df2e` on GUI
  button **`"EDC_TEST_FIGHT"`** — the Shipworks Editor `TEST` dialog (`SMALL`,
  `MID`, `LARGE`, `CARRIER` checkboxes).

---

## 6. Escadra `role` (`+0x3a8`) reference

| `role` | Meaning | Producer |
|---|---|---|
| `0` | Default / test fleet | `SomethingWithLevelSerias` test scenes |
| `1` | Trade Convoy | `spawnTradeConvoy?` `0x140209454` |
| `2` | Garrison / city-docked fleet | `Prologue_SpawnEventEscadra` `0x140203a80`, `Escadra_TryTransferToGarrison` `0x1400eebd5`, tutorials, skirmish/editor test |
| `3` | Tactical Missile Taskgroup | `0x14020e6ec` (`TASKGROUP` detachment with `aircraft == 0`) |
| `4` | Tactical Air Taskgroup | `0x14020e6ec` (`TASKGROUP` detachment with `aircraft > 0`) |
| `5` | Strike Group | `Endgame_BuildEscadra` `0x1402085e6`, prologue `"BOREY"`, dialog `"GOVERNOR"` |
| `6` | Endgame Khiva Assault Group | `spawnEndgameShips?` `0x1402624ce` (overrides `5` → `6`) |
| `7` | Player detached formation (fuel-check exempt) | `DetachShips?` `0x140201a60` |

## 7. Group-count sites

The spawn **counts** (not the ship lists) live at four loop sites; see
[group-counts.md](group-counts.md) for the full audit with guard bytes:
trade convoy count `0x14028fd63` (`MOV EDX, 0x1e` = 30), campaign SG bound
`0x14028fd4d` (`CMP ESI, 0x6`), Khiva assault wave `0x1402623de` (vector walk,
max 4), outer Nuclear War wave `0x1402629b1` (`CMP EAX, 0x3`). Tactical-group
counts are the per-city `chanceA`/`chanceB` probabilities (pool `0` only).

## 8. Entry bytes (16-byte version guards for 1.163)

| Function | Address | First 16 bytes |
|---|---|---|
| `Fleet_CreateEscadraFromShips` | `0x1401c7060` | `48 89 5c 24 10 48 89 6c 24 18 48 89 74 24 20 57` |
| `somethingSpawnShips` | `0x140203b80` | `48 8b c4 48 89 58 10 48 89 70 18 48 89 78 20 55` |
| `Endgame_BuildEscadra` | `0x1402077c0` | `48 8b c4 48 89 58 08 55 56 57 41 54 41 55 41 56` |
| `addReinforceingShips` | `0x14021b8c0` | `48 8b c4 48 89 58 18 55 56 57 41 54 41 55 41 56` |
| `endgameTarkhanShipNames?` | `0x140238e70` | `48 89 5c 24 10 48 89 74 24 18 48 89 7c 24 20 55` |
| `restOfEventsPrologueAntIsaiah` | `0x14028de70` | `48 8b c4 48 89 58 10 48 89 70 18 48 89 78 20 55` |
