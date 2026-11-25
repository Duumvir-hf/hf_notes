# Nuke mechanics in Highfleet 1.163

Status: **read-only static analysis**, 2026-11-25. Program
`Highfleet.exe.unpacked.exe v1.163` (image base `0x140000000`, Ghidra project
`HighfleetServer`), plus the live `Libraries/OL.seria`, `Libraries/parts.seria`
and `Ships/*.seria`. **No in-game run was performed for this report.** Tags:
**[code]** read in the disassembly/decompilation, **[data]** read from the
installed data files, **[game]** prior measured run recorded in kata, **[guess]**
inference.

All addresses used below were re-audited in the 1.163 program on 2026-11-25
after the shared Ghidra instance had briefly been switched to 1.151; the audit
log is `/tmp/opencode/nuke-1.163/raw/163-only-audit.txt`. The only 1.151
material in this report is the clearly labelled comparison in §7.

Raw notes, verbatim agent replies and parent verification:
`/tmp/opencode/nuke-1.163/` (index in §10).

---

## Short answer

`4e+07` in `OL.seria` is **not "damage per part"**. It is the nuclear
missile's **strategic warhead yield**, and the stock tactical path deliberately
does not feed it to the area-blast formula:

1. `m_missile_explosive=4e+07` is loaded into the missile's `Stats` record at
   `+0x168` [data/code].
2. A map-spawned nuclear missile gets creatureLogic **id 10**. On its first
   tick, id 10 reads the stat and, because it is above `100000.0`, replaces it
   with `Body+0x2dc = 100.0` and sets `Body+0x2e4 = 4` (kind 4) [code].
3. `Body_Explode` returns before any area damage for kind 4. It runs the nuke
   visual handler `0x14002bb50`, which calls `handleMissileIntercepts` on the
   linked **map** missile. That creates a strategic `Explosion` mark carrying
   the raw `4e7` [code].
4. The actual hull/city damage is applied later on the strategic map by
   `Explosion::Update` `0x140116040`, using inverse-cube formulas
   `E/(1e6·d³)`, not `E` as a radius or a kill flag [code].
5. Shipworks' **`EXPLOSIVE`** field is a *different* field: it writes the
   per-part `Body+0x2dc` (`m_explosive`) directly, with no `100000` clamp and
   without changing `Body+0x2e4` (type stays 1). When that part dies,
   `Body_Explode` runs the ordinary area branch with `E = 4e7`:
   `R = sqrt(4e7) ≈ 6324.6 m`, up to ~41.7k damage per part at the centre.
   That is why the ship/formation vaporizes there but not in the stock nuke
   path [code].

The two numbers are the same float; the **fields and code paths are not**.

---

## 1. Three fields named "explosive"

This naming collision is the root of the confusion.

| field | location | who writes it | clamp | who reads it |
|---|---|---|---|---|
| `m_missile_explosive` | `Stats+0x168` | `OL.seria` loader (`atof`) | none in the loader | `Part_GetExplosive` `0x140048870`, and direct comparisons at 1e5 |
| `m_explosive` | `Body+0x2dc` | save/ship loader `bodySecondaryInit` `0x14002694d`; Shipworks `ED_PROP_EXPLOSIVE` commit `0x14015c106` | **none** | `Body_Explode` / `Body_DieExplode` |
| `m_explosive` | `Missile+0x2f0` (strategic map missile) | `initProfileMissiles?` `0x1400ed1f9` | none | `handleMissileIntercepts` and the map mover |

The strategic `Missile+0x2f0` is registered under the property name
`m_explosive` (`Missile_RegisterProperties` `0x1401046e2`), even though it is
not the same field as `Body+0x2dc` [code]. That is why "the nuke has
`m_explosive = 4e+07`" is true on the map layer and misleading for the
tactical body.

### 1.1 `OL.seria` data

Live `Libraries/OL.seria` [data]:

| record | `m_missile_explosive` | `m_missile_ap` | `m_missile_fuze_type` |
|---|---|---|---|
| `MDL_NUKE_01/02/03/04` | **4e+07** | absent (0) | absent |
| `MDL_NUKE_01_CONV..04_CONV` | **700** | **100** | **1** (except 04) |
| `MDL_MISSILE_01` Zenith | 300 | 30 | absent |
| `MDL_MISSILE_02` Nadir | live file edited (0); `OL_og.seria` has 400 / 30 | | |
| `MDL_MISSILE_03` Sprint | 100 | 0 | absent |

`OL.seria` is **regenerated on game exit** by `App_ReloadSeria` `0x140240b90`,
which serialises `ObjectsLibrarySingleton` to `Libraries/OL.seria` [code]. The
live file is not pristine (`MDL_MISSILE_02` has lost its warhead/AP); use
`OL_og.seria` as the reference. The nuke/CONV rows are identical in both [data].

`m_missile_explosive` is registered as a float property by `populateObjects`
`0x1402821c0` and parsed by `fProperty_ReadField` `0x14018ff40` as plain
`atof` + `(float)`; `4e+07` becomes `0x4C189680` = `40000000.0f` [code]. There
is no clamp in the loader.

### 1.2 `parts.seria` nuke bodies

All eight nuke Body prototypes (`MDL_NUKE_01..04` and `_CONV`) have 20 HP and
**no `m_explosive` and no `m_explosive_type`** [data]. A freshly placed nuke
part therefore starts at `Body+0x2dc = 0`, type 1. A nuke missile destroyed
before its logic arms does **not** explode at all (`Body_DieExplode` sees
`E = AP = 0`) [code].

---

## 2. Ordinary tactical blast formula (the path that vaporizes)

`Body_Explode` `0x140029b60` [code]:

* Entry gates: `(E != 0 || AP != 0) && byte[+0xec] != 0`; kind 4 exits early.
* `S = 2·sqrt(E)`, `R = S/2 = sqrt(E)`.
* Owner factor `k`: `1.0` if the exploding body's owner is the player ship,
  otherwise `0.66` (hardmode can force `0.66` even for the player).
* Per victim:
  `d_edge = max(0, dist(centre, victim) - victim.radius)`;
  if `d_edge >= R` → no damage; else
  `damage = floor(5 · scale · (1 - d_edge/R)²)`, where `scale` is
  `S_eff = S·k` (or `visibility·S_eff` for area candidates, with armour
  shadowing in the 360-bucket table).
* `CVTTSS2SI` converts to **signed 32-bit** at `0x14002cf6c`; no clamp.
* Chain: `budget = min(1, +0x2c0 + 0.5) · AP`; the directly hit armoured body
  takes `budget·4.0`; armour costs `-10.0`; an ordinary part costs
  `maxHP/20.0`; category `0x80` (truss/hardpoint) parts are free.
* Kind 4 (`Body+0x2e4 == 4`) calls `0x14002bb50` and returns: **no area
  damage, no ApplyDamage**.

### 2.1 Numbers for a raw `E = 4e7`

| quantity | `k = 0.66` | `k = 1.0` |
|---|---|---|
| radius `R = sqrt(E)` | **6324.56 m** | 6324.56 m |
| centre damage per part | **41 742** | **63 245** |
| damage at half radius | 10 435 | 15 811 |
| 400-HP part (toughest in game) killed within | **~5 705 m** | ~5 822 m |
| same, player ship ×0.6 | ~5 525 m | — |

Every part of every ship within ~5.7 km dies on the first frame; a 100-300 m
ship is entirely inside the blast. This is **not** an int32 overflow: the
overflow thresholds are ~1.8e15 to ~1.1e17, and above them the signed
conversion yields INT_MIN and `Body_ApplyDamage` *heals* the part because it
has no negative-damage check [code/arithmetic].

For comparison, the clamped value `E = 100`, if it were run as an ordinary
blast, gives `R = 10 m`, centre 66-100 per part — a small pop, not a nuke.

---

## 3. The stock nuclear missile path (creatureLogic id 10)

1. `SCR_MISSILE_SpawnDefenders` `0x1402d45c0` builds the tactical battle and
   spawns one body per carried missile. `SCR_MISSILE_AssignDefender`
   `0x140263700` sets logic id **1 for a `_CONV` OID, 10 otherwise**; if the
   map missile is at least 100 units from its stored target point, even a
   `_CONV` is forced to **10** (`0x140263931`) [code].
2. Logic 10, state 0 (`0x1400383cf`), first tick [code]:
   * `Body_StoreHealthClamped(10.0)` at `0x140038407` (this sets the spawned
     tactical missile's HP to 10; it is **not** damage);
   * `E = Part_GetExplosive()` at `0x140038410`;
   * `Body+0x2dc = E` at `0x140038415`;
   * `COMISS E, 100000.0` at `0x14003841d`; if above:
     `+0x2dc = 100.0` (`0x140038426`), `+0x2e4 = 4` (`0x140038435`);
   * `INC [+0x3cc]` at `0x14003875c`, state 1.
3. Logic 10, state 1 (`0x140038026`) flies the missile. It calls
   `Body_DieExplode` when
   `|body.pos - target.+0x1840| < 400.0`, or when `|Δx| < 200.0`
   (`0x14003812c`/`0x140038151`) [code].
4. `Body_DieExplode` `0x140023ac0` calls `Body_Explode` at `0x1400240d8` only
   if `(E != 0 || AP != 0) && +0xec != 0`; for kind 4 that becomes
   `0x14002bb50` → `handleMissileIntercepts(linked map Missile, 0, 1.0)`.
5. `handleMissileIntercepts` `0x1400f11b0` creates the strategic `Explosion`
   mark named `BLAST` with the map missile's raw `+0x2f0 = 4e7`, sets
   `isNuclear`, creates trigger `FIRST_NUCLEAR` and raises
   `Profile.escalation` to at least 1 [code]. **No tactical area damage.**

The tactical screen therefore shows the nuke visuals and the
`suspend_rising` arrival event; the defending ships' HP is unchanged at the
detonation frame. The measured campaign loss appears **after** the battle, when
the strategic mark is processed [game, kata `yvae`].

If the missile is destroyed before state 0 arms it, `+0x2dc = 0` and there is
no explosion at all [code/data].

---

## 4. The conventional `_CONV` path (creatureLogic id 1)

`_CONV` records are not nuclear: `700` explosive, `100` AP, fuze type 1
[data]. Logic 1/3/4 has **no 100000 clamp**: `0x14003731f` reads the stat and
stores it raw into `+0x2dc`; it also stores AP into `+0x2e0` and resets health
to `m_launched_health` [code].

For a stock `_CONV` (700, fuze 1):
* the fuze path spawns `(int)(700/100) = 7` bomblets, then sets
  `E = max(700/5, 10) = 140`, giving `R = 11.8 m`, ~78/part (k=0.66) plus the
  AP-100 chain (budget 50, first armoured hit 200);
* a contact/state-4 death would use `E = 700` (`R = 26.5 m`, ~175/part).

It never sees 4e7 and never takes kind 4. **But** if a `_CONV` missile's stat
is edited to `4e7` and it spawns as logic 1 (within 100 units), the raw 4e7
reaches the ordinary blast — see §6.

---

## 5. Strategic map damage (what the 4e7 actually drives)

`Explosion::Update` `0x140116040` (Explosion vtable `+0x160`, called by the map
tick `0x140229a80` for every `Profile->explosions` entry) [code]:

* `life += dt`; `radius += (1000/life + 340)/1000 · dt` — **the ring radius is
  warhead-independent** (about 24.5 map units after 60 s; ~26 at 66 s). Only
  the intensity scales with `E`. Damage is applied when the expanding ring
  crosses an object, and only for `isNuclear` marks with intensity > 0.1, so
  the nuke must be close to its target.
* Fleet: per-part HP loss `min(cap, E/(1e6·d³))`, with `cap = 999` for AI ships
  and `50` for player ships; player-fleet distance is floored at 2.0. At
  `d = 1`, a nuke gives **40 per part** on an AI ship; the player's 5 per part
  comes from the 2.0 distance floor (40 is below the 50 cap).
* City: `deaths = min(pop, pop^1.1 · (E/1000/d³) / 300000)`. Below 60 % of the
  initial population: `radiation = 100`, `damaged = 1`, `aad = 0`; below 30 %:
  production and city bonus cleared; city items are destroyed with probability
  `deaths/pop`. A 1M city hit at `d ≈ 1` loses ~53 % → ruined.
* Hostile map missile: integrity `-= E/(1e6·d³)` per substep, clamped at 0
  (40 at `d = 1` for 4e7).
* Aircraft group: `E/(3e5·d³)` per craft, multiplied by `rand(0.5..2.0)` for a
  nuclear mark. The "<1.0 core radius → at most 3 craft" branch exists but is
  dead in practice: both callers pass core radius 1.0, and conventional marks
  skip the whole `isNuclear` damage block, so conventional map explosions deal
  no aircraft damage.
* Nuclear branch: `FIRST_NUCLEAR` trigger, `escalation = max(escalation, 1)`;
  the `NUCLEAR_ESCALATION_EXPLOSION` dialogue and the endgame `NUCLEAR_WAR`
  path read those.

Measured campaign result (hf_ally v1.3.29, `Highfleet_2026-11-25_01-45-05.log`)
[game]: at the kind-4 detonation the hull is unchanged; by the next spawn each
defending ship has lost **11-24 parts and 1.8k-2.9k hull**, with the ships
surviving. That matches the per-part `40` at close range; it is not an
unconditional fleet wipe.

---

## 6. Why "40 000 000 in Shipworks" vaporizes ships

The vanilla editor exposes a text row **`ED_PROP_EXPLOSIVE`**, caption
`"EXPLOSIVE"`, built by `somethingShipworks?` `0x1400c7400` [code]. The row
displays `Body+0x2dc` (`0x14014dbee`) and the text-commit dispatcher
`rep_14014f300_New` `0x14015a600` compares the 17-byte name at
`0x14015c088..0x14015c0b3` and executes:

```asm
0x14015c100  CALL qword [RAX+0x2f8]   ; Gui_GetNormValue -> (float)atof(text)
0x14015c106  MOVSS [RBX+0x2dc], XMM0  ; Body.m_explosive := typed value
```

There is no clamp and no type change; `Body+0x2e4` stays at its default
**1** [code]. When that part is destroyed:

* `Body_DieExplode` `0x140023ac0` → `Body_Explode` `0x140029b60`;
* `E = 4e7`, kind 1 → ordinary area branch;
* `R = sqrt(4e7) ≈ 6324.6 m`, centre damage `41 742` (k=0.66) / `63 245`
  (k=1.0) per part;
* every part of every ship within ~5.7 km dies on the first frame.

The stock nuclear missile never does this because its 4e7 is read from
**Stats** and passes through the logic-10 clamp (`100.0`, kind 4) before
`Body_Explode`. The editor value is the part's **death blast**, not the missile
warhead stat.

If a `_CONV` missile's *Stats* value is raised to 4e7 instead, the result
depends on its logic id: logic 10 clamps it; logic 1/3/4 does not, so a
close-spawned `_CONV` can also deliver the raw 4e7 blast.

The Shipworks **Missile Defense** sandbox in `hf_ally` is a separate matter:
before v1.3.30 it fed the 4e7 yield into the native kind-4 continuation and
destroyed the whole formation (kata `csb9`); v1.3.37 uses a campaign-matched
per-ship HP budget (the inverse-cube `min(cap, E/(1e6·d³))` at the campaign
fleet distance, cap 999/50, distance floor 1.0/2.0, applied per part and spread
across the targeted fleet's hull, HP subtracted directly and the part kill slot
called like `0x1400e6aa0`, master/root part excluded, capped at 2,500 hull
scaled by part count). That is a mod design choice, not
vanilla behaviour.

---

## 7. Version differences (1.151 vs 1.163)

* The tactical type-4 branch and the 100000 clamp exist in **both** versions:
  1.163 `Body_Explode` `0x140029b60` / branch `0x140029bf9` / clamp
  `0x14003841d..0x140038435`; 1.151 `0x140028520` / branch `0x1400285d1` /
  clamp `0x140036f36..0x140036f4e` [code].
* `m_explosive` → `Body+0x2dc` is raw in both (1.151 `0x14002532d`; 1.163
  `0x14002694d`) [code].
* **Real difference:** 1.151's `ConfigureStrategicMissileForTacticalBattle`
  `0x140241680` had an escalation-dependent `_CONV` ↔ nuke part swap
  (`escalation == 0` converts a non-`_CONV` attacker to its `_CONV` twin;
  `escalation >= 1` strips `_CONV` from a conventional attacker). 1.163's
  `0x140263700` has no such block [code]. Whether the swap moved elsewhere in
  1.163 is open.
* 1.163 `_CONV` data changed 500 → 700 and gained `m_missile_ap = 100`
  [data-from-note; the 1.151 file is not installed].

---

## 8. Corrections to the older notes

* `CALL [RAX+0x200]` at `0x140038164` is **`Body_DieExplode`**
  (`0x140023ac0`), not `Body_Explode`. `Body_Explode` is called from inside it
  at `0x1400240d8`.
* `CALL [RAX+0x208]` at `0x140038407` is **`Body_StoreHealthClamped(10.0)`**,
  not `Body_ApplyDamage(40.0)`.
* The 100000 clamp has **one owner**: logic 10, state 0
  (`0x140038410..0x140038435`). Logic 1/3/4 and logic 2 have unclamped
  `Part_GetExplosive` stores.
* The area blast needs the extra entry gate `byte[+0xec] != 0`.
* The `×4.0` rule applies to the **directly hit armoured body** (`+0x3b0`), not
  to "the first armoured chain part"; armoured chain candidates take plain
  budget.
* The armour threshold `Stats+0xc4` is not applied to blast/chain damage (all
  those call sites pass the threshold flag 0).
* `max(R, 200)` only widens the candidate scan; the falloff itself stays gated
  at `d_edge < R`.
* The deferred blast record stores `S_eff = 2·sqrt(E)·k` at `+0x10`, not `E`.
* `Missile+0x2f0` is also named `m_explosive` in the property table; it is not
  `Body+0x2dc`.

---

## 9. Open questions

1. The exact screen name the user calls "Shipworks" — the `ED_PROP_EXPLOSIVE`
   commit path is proven, but whether it is `EditorExtended` or the campaign
   shipworks is [guess].
2. `Explosion+0x261 isShipDamaged` writer — not found; probably not the damage
   carrier.
3. The ship-removal condition inside `0x1400e6aa0`/`0x1400f0cc0` — the measured
   campaign result shows ships surviving with 11-24 parts lost; the exact
   "add ship to kill list" condition is only partly pinned.
4. Whether a `_CONV` dies by contact/timer (E=700) or by the fuze path
   (E=140) in practice.
5. Meaning of the `+0xec` arming byte.
6. Whether 1.163 moved the 1.151 escalation-dependent `_CONV` swap elsewhere
   (candidate: `SCR_MISSILE_SpawnDefenders` `0x1402d45c0`).
7. `OL.seria` is a runtime snapshot; keep `OL_og.seria` as the reference.

---

## 10. Raw reports

All under `/tmp/opencode/nuke-1.163/`:

* `blackboard.md` — task brief, addresses, hypotheses.
* `agent-d1-data.md` — data + property layer, editor path (552 lines).
* `agent-d2-logic.md` — `creatureLogic` logic 1 vs 10 state machine (463 lines).
* `agent-d3-strategic.md` — strategic `Explosion::Update` formulas (569 lines).
* `agent-g1-math.md` — independent blast-math verification (409 lines).
* `agent-g2-crosscheck.md` — independent cross-check and 1.151 comparison
  (375 lines).
* `raw/` — verbatim agent replies, parent verification notes, and the
  program-switch warning.
* `README.md` — index.
