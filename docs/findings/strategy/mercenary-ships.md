# Mercenary, Tarkhan, and reinforcement ships: how each system works (1.163)

Program: `Highfleet.exe.unpacked.exe` v1.163, image base `0x140000000`.
Re-audited 2026-11-12 from `Npc_RegisterProperties` (`0x14023fbd0`),
`somethingHireShips?` (`0x140266100`), `!ClickFunc` (`0x140136a75`),
`Prologue_SpawnEventGroups` (`0x1402a7f60`), `spawnEndgameShips?` (`0x140261a20`),
`restOfEventsPrologueAntIsaiah` (`0x14028de70`), and `addReinforceingShips`
(`0x14021b8c0`). Tags: [code].

## Short answer

The earlier 2026-10-30 note conflated city mercenary ships with
`addReinforceingShips`. In reality, HighFleet has **three separate mechanisms**:

1. **Hireable mercenary / endgame free ships in cities (`FOR_HIRE` / `FOR_JOIN`):**
   Stored in `Profile->npcs` (`+0x2c0`, serialized as `m_npcs`) as anonymous `NPC`
   records with `m_tarkhan == 0`, `m_location == <city name>`, and
   `m_ship_first == <design name>`.
   * **Campaign mercenary cities** (`Location+0x25c == 6`): stocked at worldgen
     by `Prologue_SpawnEventGroups` (`0x1402a8342`) with up to **5** ships rolled
     from `somethingSpawnShips` **pool 2** (28 pushes / 27 unique designs,
     budget `citysize / 20.0`).
   * **Endgame cities** (all non-Khiva cities during Nuclear War): wiped and
     restocked by `spawnEndgameShips?` (`0x140262bda`) with up to **3** cheapest
     ships rolled from `somethingSpawnShips` **pool 4** (14 designs), shown as
     free `FOR_JOIN` cards because `Profile.endWar != 0`.
2. **Named Tarkhan personal ships (`FOR_TARKHAN`):**
   `NPC` records with `m_tarkhan == 1` (`+0xf9`). Rendered on the same city pier
   strip (`somethingHireShips?` `0x140266100`), but clicking starts `HELLO_<NAME>_1`
   diplomacy (`!ClickFunc` `0x140136a75`) instead of buying. Their personal ship
   (`m_ship_first`) is rolled at campaign start (`0x14028f620`) from an **11-ship
   switch**, and joins your fleet on `DIPDIALOG_WIN`.
3. **Call-in reinforcement / scripted ships (`addReinforceingShips` `0x14021b8c0`):**
   Hard-coded tier tables (`tier 0` = 5 ships, `tier 1` = 8 ships, `tier 2` =
   unused partial-double of tier 1) that instantiate a ship directly into a
   container. Used by Tarkhan support button **`ASK_REINFORCE`** (`tier 1`),
   `DIPDIALOG_WIN` fallback (`tier 1`), dialog `JOIN=<name>,<design>`, and the
   Khiva finale `ENDWAR_CARD` (`tier 0` + `endgameTarkhanShipNames?` `0x140238e70`).

---

## 1. `NPC` record layout (`Npc_RegisterProperties` `0x14023fbd0`, size `0x158`)

| Offset | Serializer key | Type | Role |
|---|---|---|---|
| `+0x48` | `m_loyalty` | `i32` | Star count = `m_loyalty / 3` |
| `+0x50` | `m_name` | `EscadraName` | Identifier (`OMAR`, `FASIL`, or numeric string for merc ships) |
| `+0x70` | `m_fullname` | `EscadraName` | Random two-part display name |
| `+0x90` | `m_description` | `EscadraName` | `HELLO_DESCRIPTION_<female>_<age>` |
| `+0xb0` | `m_location` | `EscadraName` | **City name** where the card/ship appears |
| `+0xd0` | `m_ship_first` | `EscadraName` | **Ship design name** spawned at the city pier |
| `+0xf0` | `m_ship_first_seed` | `i32` | RNG seed (`0..999`) applied around `CalculateStats` |
| `+0xf6` | `m_joined` | `u8` | Set to `1` when recruited (`DIPDIALOG_WIN` or campaign start for `PETR`/`DAUD`/`FASIL`/`HARISHID`) |
| `+0xf9` | `m_tarkhan` | `u8` | **`0` = hireable merc ship (`FOR_HIRE`/`FOR_JOIN`); `1` = Tarkhan (`FOR_TARKHAN`)** |

---

## 2. City Pier Strip (`somethingHireShips?` `0x140266100` & `!ClickFunc` `0x140136a75`)

* On landing in `CURRENT_LOCATION`, `somethingHireShips?` filters `Profile->npcs`
  by `npc->m_location == location->name` (`+0xc8`), instantiates a live preview
  `Creature` of `npc->m_ship_first` (`0x140209b00`) parked at a free pier
  (`PIER_02` / `PIER_03`), computes `price = floor(ship->m_tele_price / 100) * 100`
  (`+0x15d0`), and creates a `LANDMODE_NPC` card:
  * `m_tarkhan == 0`: `"for_hire"` with price (or `"for_join"` free if `Profile.endWar != 0`).
  * `m_tarkhan != 0`: `"for_tarkhan"`.
* Clicking `LANDMODE_NPC` (`0x140136a75`):
  * If `m_tarkhan != 0 && endWar == 0`: opens `HELLO_<NAME>_1` Tarkhan diplomacy.
  * Otherwise (`FOR_HIRE` or `endWar != 0`): deducts `price` from `Profile.m_gold`
    (`+0x2a8`) when `endWar == 0`, links the already-spawned pier `Creature`
    directly into the player's escadra (`SetLink(ship+0xa68, playerEscadra)`),
    shows `NEW_SHIP`, and plays `sold_01`. No mercenary flag remains on the ship.

### Pools feeding city hire cards

* **Campaign Mercenary Cities (`somethingSpawnShips` pool 2, 28 pushes / 27 unique):**
  `Yars, Yars Mk2, Mockingbird, Skylark, Tanker, Rook, Lightning, Fenek,
  Navarin, Scarab, Meteor Mk3, Fatman, Flower, Intrepid, Fearsome (x2), Jaguar,
  Paladin Mk2, Gladiator, Gepard, Archangel, Kormoran, Negev, Voskhod, Wasp,
  Longbow, Egida Mk2, Triumphant`.
* **Endgame Free City Ships (`somethingSpawnShips` pool 4, 14 designs):**
  `Tanker, Fenek, Scarab, Fatman, Intrepid, Paladin Mk2, Gladiator, Fearsome,
  Gepard, Archangel, Kormoran, Negev, Voskhod, Wasp`.

---

## 3. Named Tarkhan Ships

* **New-profile cast (`saveEventsToProfile?` `0x1402a67c3`–`0x1402a6ead`):**
  Creates 16 `NPC` records. Nine have `m_tarkhan = 1` (`BOY, GIRL, HARISHID,
  HISSID, KNIGHT, SCOUT, AIRBORNE, VETERAN, OMAR`); `HARISHID` overrides
  `m_ship_first = "Gladiator"` (and starts joined in the main campaign),
  `GOVERNOR` gets `"Varyag"`, and the rest default to `"Paladin Mk2"`.
* **Campaign city placement (`restOfEventsPrologueAntIsaiah` `0x14028f570`–`0x14028f788`):**
  Unjoined Tarkhans (`m_tarkhan != 0 && m_joined == 0`) get a city and roll
  `m_ship_first` uniformly via `RandInt(1, 11)`:
  1. `Intrepid Mk2`
  2. `Jaguar Mk2`
  3. `Paladin Mk2`
  4. `Gladiator`
  5. `Gepard`
  6. `Gepard Mk2`
  7. `Archangel`
  8. `Wasp`
  9. `Longbow`
  10. `Kormoran`
  11. `Negev`
* **Endgame Khiva Tarkhan Flagships (`endgameTarkhanShipNames?` `0x140238e70`):**
  4 groups (`14 / 6 / 4 / 3` designs) picked by `creatureLogic` id `10500`
  (`0x1400430df`) to reinforce the fleet at `KHIVA`; see [enemy-ship-pools.md](enemy-ship-pools.md).

---

## 4. `addReinforceingShips` (`0x14021b8c0`)

```c
void addReinforceingShips(
    Creature    *container,
    EscadraName *extraName,
    int          tier,
    char         showNewShipMsg,
    char         playCheer,
    EscadraName *tarkhanName
);
```

If `extraName` is non-empty, spawns `extraName` without rolling; otherwise rolls
from `tier`:

* **tier 0** (`RandInt(0, 4)`, 1 ship):
  `Navarin`, `Scarab`, `Meteor Mk3`, `Fatman`, `Lightning Mk2`
* **tier 1** (`RandInt(0, 7)`):
  indices `0..2` (`Navarin`, `Intrepid`, `Jaguar`) spawn **two** copies; indices
  `3..7` (`Voskhod X2`, `Triumphant`, `Longbow`, `Vega`, `Kormoran`) spawn **one** copy!
* **tier 2** (`RandInt(0, 7)`, unused in 1.163):
  identical to tier 1 (`Navarin`/`Intrepid`/`Jaguar` x2, `Voskhod X2`..`Kormoran` x1).

### Callers of `addReinforceingShips`

| Caller | Site | Args / Gameplay meaning |
|---|---|---|
| `Dialog_ExecuteChoiceActions` (`ASK_REINFORCE`) | `0x1400b4b69` | `(PLAYER_ESCADRA, "", tier=1, 1, 1, "")` — spending Tarkhan stars (`AIRBORNE` / `KNIGHT` / `DAUD` support action) to request a reinforcement ship. |
| `Dialog_ExecuteChoiceActions` (`DIPDIALOG_WIN`) | `0x1400b4f14` | `(PLAYER_ESCADRA, npc->m_ship_first, tier=1, 1, 1, npc->m_name)` — fallback when a recruited Tarkhan has no live pier ship object. |
| `Dialog_ExecuteChoiceActions` (`JOIN`) | `0x1400b5316` | `(PLAYER_ESCADRA, <design>, tier=1, 1, 1, <prefix>)` — explicit `<prefix>,<design>` dialog reward (`extraName` non-empty, so `tier=1` is bypassed). |
| `creatureLogic` id `10500` (`ENDWAR_CARD`) | `0x1400432a9` | `(khivaFleet, endgameTarkhanShipName, tier=0, 0, 0, panelName)` — spawns the chosen Khiva flagship (`extraName` non-empty; note `tier=0` is only rolled if `endgameTarkhanShipNames?` returned empty!). |
