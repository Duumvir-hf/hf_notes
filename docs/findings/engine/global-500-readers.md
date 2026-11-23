# The "500 global": the shared `500.0f` literal and its 85 direct readers (Steam 1.163)

Editing one value in Cheat Engine on 1.151 showed three visible effects at once: a different
engine overheat threshold, enemy pre-fire gun laser length, and rocket acceleration.

**It is not a gameplay variable.** `0x140352838` is the only 4-byte-aligned `500.0f` literal in
the Steam 1.163 `.rdata` section. **85 instructions** in `.text` read it directly. Editing it
affects all 85 readers; to decouple the behaviors, redirect only the intended instructions.

* 1.151 counterpart: `0x1403263f4` (`00 00 FA 43`), with 77 matching instructions indexed in Ghidra
  (no independent full sweep for that build). The 1.151 site `0x140034230` (inside `creatureLogic`) is
  `F3 0F 59 0D BC 21 2F 00` = `MULSS XMM1, [0x1403263f4]` — one of the pool's many readers, not the
  global itself.
* 1.163 pool neighbourhood (all `f32`):
  `0x140352830` 300.0, `0x140352834` 400.0, **`0x140352838` 500.0**, `0x14035283c` 600.0,
  `0x140352840` 700.0, `0x140352844` 1000.0.
* Every reader listed below is a plain `OP xmm, dword ptr [0x140352838]` with the RIP-relative disp32
  as the **last 4 bytes** of the instruction, so a site can be retargeted at a mod-owned float by
  rewriting only those 4 bytes. The signed 32-bit displacement is measured **from the end of that
  instruction**; a nearby allocation avoids relying on the DLL being in range.

Method: Ghidra's `search_instructions operand_pattern=140352838` and `get_xrefs_to 0x140352838`
both return 82 readers. A linear `.text` disassembly of the unpacked Steam 1.163 image finds those
same 82 plus `0x14003a253`, `0x14003a664` and `0x14022eca2`. The bytes at all three extra sites
match in the Ghidra program despite their absence from its search and xref results.

```sh
objdump -d -Mintel -j .text Highfleet_1.163_dump.exe | grep -E '# 0x140352838([[:space:]]|$)'
```

## The three features, and the sites that belong to them

| feature | sites | instruction | what it does |
|---|---|---|---|
| **engine overheat trip** | `0x14002861c` `Engine_ApplyThrustForce` | `COMISS XMM0, [500]` | If the engine temperature at `part+0x2ec` is at least 500, set `part+0x810` to 3.0 seconds and return without applying force. `Part_Tick` counts down the lockout. See [engine heat](../physics/player-thrust-keys-and-engine-heat.md). |
| **enemy pre-fire gun laser** | `0x14029d538`, `0x14029d540`, `0x14029d82a`, `0x14029d832` `BattleHud_DrawCombatMarkers` (`0x14029c5c0`, called from `combatEvents`) | `MULSS XMM6/XMM7, [500]` (two pairs) | end point of the per-gun aim line: `end = gunWorldPos + unit(aimPoint - hull) * 500`, drawn with `Gfx_DrawWorldLine` in red `0xFFDD2727` (cyan `0xFF00E4FF` for calibre logic 160, yellow `0xFFFFDE00` when the ammo flag `+0x164` is set). Gated on `0.0 < creature+0xad4` being **false**, i.e. **these lines are drawn for enemy ships only** — the "pre-fire laser". Two loops = gun groups 0 and 1 (`+0x13a8..13b0`, `+0x13c0..13c8`). |
| **rocket acceleration** | `0x1400348f2` (logic 130), `0x140034c0c` (logic 132) `creatureLogic` | `MOVSS XMM7/XMM9, [500]` | While the burn timer `+0x3d4` is positive, apply `mass * 500` along the body heading via vtable `+0x178` ([`Body_AddForceAtCenter`](../physics/static-engine-force-path.md)). For logic 130, the mass load is at `0x140034943`, the multiply at `0x14003494b` and the call at `0x140034968`. |
| **rocket path preview** | `0x1402a02d6`, `0x1402a02e4`, `0x1402a04c4`, `0x1402a04d1` `BattleHud_DrawCombatMarkers` | `MULSS XMM13/XMM14/XMM6, [500]` | Two pairs scale the per-step thrust contribution to the dashed trajectory. Both loops use `1.5 s` from `0x1403527cc`, a step of `1 / CORE_FRAMERATE`, and a separate `.05` coefficient at `0x14035467c` in the velocity multiplier, not a `9.82` gravity literal. |

These rows cover 11 readers. The full table below also includes a separate rocket lead
calculation in `AiWeaponController`.

## All 85 direct readers

The `OP` column records the instruction mnemonic. Outside the traced paths, descriptions based
on the containing function or surrounding instructions are provisional.

### combat / physics / ordnance

| site | function | OP | what it does |
|---|---|---|---|
| `0x14002861c` | `Engine_ApplyThrustForce` | COMISS | **overheat trip** (see above) |
| `0x1400348f2` | `creatureLogic` | MOVSS | **rocket thrust, logic 130** (cluster rocket, 1.5 s burn) |
| `0x140034c0c` | `creatureLogic` | MOVSS | **rocket thrust, logic 132** (300 mm rocket) |
| `0x14003a253`, `0x14003a664` | `creatureLogic` | MOVSS ×2 | two state branches pass `500 / value` to `0x14001c7b0`; the exact gameplay role is not traced |
| `0x1400448e9` | `creatureLogic` | SUBSS | missile-warning GUI element: `field +0x154 - 500` alongside `field +0x150 - 700` (`0x140352840`), before setup of `gui_std_click`/`gui_std_focus` and `flash_large_shadow` |
| `0x14029d538`, `0x14029d540`, `0x14029d82a`, `0x14029d832` | `BattleHud_DrawCombatMarkers` | MULSS ×4 | **enemy pre-fire gun laser length** (see above) |
| `0x14029e943` | `BattleHud_DrawCombatMarkers` | MOVSS | third float argument to `renderGuiFlashes?` `0x140243cf0` (the HUD's per-body flash/glow renderer) — a size/scale, not physics |
| `0x14029f1d1` | `BattleHud_DrawCombatMarkers` | MOVSS | thruster glow: `intensity = thrust_field / 500` summed over the ship's thrusters (the `thrusters` loop at line ~1344 of the decompile) |
| `0x1402a02d6`, `0x1402a02e4`, `0x1402a04c4`, `0x1402a04d1` | `BattleHud_DrawCombatMarkers` | MULSS ×4 | **rocket path preview acceleration** (see above) |
| `0x1401c7716` | `FX_SpawnExplosionDebris` | MOVSS | debris effect; exact use not traced |

### AI

| site | function | OP | what it does |
|---|---|---|---|
| `0x140063857`, `0x140063860` | `AiShipTick` | MULSS x2 | x/y components of a point at 500 units along some direction (a waypoint / dodge offset) — not read further |
| `0x140063b90` | `AiShipTick` | MOVSS | `500 / (dist + k)` — an inverse-distance factor; not read further |
| `0x140063c75`, `0x140063c8c` | `AiShipTick` | MOVSS | two branches loading 500 into the same local; not read further |
| `0x140063ed4` | `AiShipTick` | MOVSS | not read |
| `0x140064812` | `AiShipTick` | MOVSS | not read |
| `0x140064e56` | `AiShipTick` | MOVSS | not read |
| `0x14006cf63` | `AiShip_SteerController` | MOVSS | 500 loaded as a max value next to `[RDI+0xd78]` and pushed as a stack argument to a vtable call — a clamp on the steering output, not read further |
| `0x14007b81a` | `AiWeaponController` (`0x14007a8d0`) | MOVSS | the group-1 branch for loaded guns with ammo `Stats+0x144 == 0x34` (calibre 52, 220 mm rocket) passes acceleration 500 and burn duration 1.5 s to the intercept solver at call site `0x14007b852` (see §9 in [gun aiming and fire logic](../combat/guns-aim-reload-fire.md)) |
| `0x14006c24d` | `rep_140068980_...atan2f` (misnamed; an AI/rhetoric function) | ADDSS | 500 added to an angle/offset — not read further |
| `0x14006c611` | `Creature_TryDropBomb` | MOVSS | bomb-drop test offset — not read further |

### aircraft

| site | function | OP | what it does |
|---|---|---|---|
| `0x14000871f` | `Aircraft_CollectAvoidanceObstacles` | MOVSS | avoidance radius/weight |
| `0x14000c40d` | `AircraftLogic_Mode3_Bomber` | MOVSS | bomber-mode distance/altitude constant |
| `0x14000d680` | `AircraftLogic_Mode6_Default` | MOVSS | default-mode constant |
| `0x14000eda7` | `AircraftLogic_SetState` | MOVSS | state-entry constant |
| `0x14000f5f2` | `AircraftLogic_LaunchAAMissile` | COMISS | rejects range at or below 500; this is the lower limit, not the upper limit |

The other four aircraft roles are not traced beyond the instruction and function name.

### battle / campaign spawn plumbing

| site | function | OP | what it does |
|---|---|---|---|
| `0x1400ed4fc` | `initProfileMissiles?` | MOVSS | strategic-layer missile inventory init (a 500-scale) |
| `0x1400f0698` | `Escadra_UpdateEscortOffset` | MOVSS | escort formation offset scale |
| `0x1400f5ead` | `StrategicFleet_SelectPursuitTarget` | COMISS | pursuit range test on the map |
| `0x14011a8df` | `Escadra_UpdateContamination` | MOVSS | contamination spread scale |
| `0x14027b935` | `addShipsToSilentStrikeGRP?` | MOVSS | strike-group layout |
| `0x1402c9d42` | `SCR_COMBAT_SpawnPlayerShip` | MOVSS | battle-start offset (spawn / camera) |
| `0x1402ce656` | `SCR_COMBAT_ReturnToMap` | MOVSS | return-to-map layout |
| `0x1402cec87` | `SCR_COMBAT_SKIRMISH_ReturnShips` | MOVSS | skirmish return layout |
| `0x1402cfd90` | `SCR_CRAFTS_VS_CRAFTS` | MOVSS | craft-vs-craft mode layout |
| `0x1402d4d15` | `SCR_MISSILE_SpawnDefenders` | MOVSS | missile-defence spawn spread |
| `0x1402d6f2d` | `deployFromReserve` | MOVSS | reserve deploy offset |
| `0x1402d71d3`, `0x1402d71db` | `Battle_SpawnQueuedShipForSlot` | ADDSS/SUBSS | queued-ship slot x/y nudge (500 either side) |
| `0x1402ebcdd` | `Ground_SpawnAlongPath` | MOVSS | ground-unit path spawn spacing |
| `0x1402f1486`, `0x1402f1b0d`, `0x1402f281c`, `0x1402f2e71` | `generateCityLandscape` | SUBSS/MOVSS | city/terrain generation distances |

The precise uses in this group are not traced.

### screens, maps, GUI, editors

| site | function | OP | what it does |
|---|---|---|---|
| `0x1400b00fc` | `mainDialogWithChoices` | ADDSS | dialog/text layout |
| `0x1400c753f`, `0x1400cc684`, `0x1400cdd14`, `0x1400ce1ce`, `0x1400cee04` | `somethingShipworks?` | MOVSS | shipworks editor view/panel math |
| `0x1400d95d2` | `displayinShipworks?` | MULSS | editor zoom/scale |
| `0x140108b32` | `displayAircraftStrategicMap` | MOVSS | map marker scale |
| `0x140138af5` | `!ClickFunc` | MOVSS | main click handler, layout constant |
| `0x140146b73` | `setupPropsSecondaryMapModes` | DIVSS | map-mode divisor |
| `0x1401a8b36` | `!FrameFunc` | MOVSS | per-frame layout |
| `0x1401c4915` | `Gui_BuildBayonetsPanel` | MOVSS | panel layout |
| `0x1401c6591` | `Dialog_BuildBriefingPanel` | MOVSS | briefing text layout |
| `0x1401d96db` | `setFontMaxSize?` | SUBSS | font metric |
| `0x1401dbf1a` | `Landing_BuildPrelandingPanel` | MOVSS | landing panel layout |
| `0x1401fd354` | `secondaryPrecombatButtons` | MOVSS | pre-combat button layout |
| `0x140203034` | `rep_1401e8a70_VCRUNTIME140::memmove` (misnamed) | MOVSS | GUI copy path, layout constant |
| `0x14021464d` | `rep_1401f85c0_HEAP::free` (misnamed) | MOVSS | GUI object teardown, layout constant |
| `0x140215489` | `rep_1401f9490_CRT-MATH::atan2f` (misnamed) | MOVSS | GUI/aim helper |
| `0x140229b81` | `renderStuffToMap?` | MOVSS | map render scale |
| `0x14022eca2` | `variousHUDDisplays?` (`0x14022c980`) | DIVSS | divides a `+0x1628` field by 500 in the calculation of a displayed value next to the localized `TON` label |
| `0x140236ae5` | `creditsScreen?` | MOVSS | credits scroll |
| `0x14026bec2` | `UpdateZoom` | MOVSS | zoom limits |
| `0x140287006` | `SomethingPreTask` | MOVSS | per-frame layout |
| `0x1402a0980` | `handleLandingGUI?` | MOVSS | landing HUD layout |
| `0x1402b5f66`, `0x1402ba377`, `0x1402be726`, `0x1402c066b`, `0x1402c0c44` | *(no function defined — undefined code inside the scene functions)* | MOVSS | scene/GUI code, not read |
| `0x1402c6d07` | `SomethingWithLevelSerias` | MOVSS | level-seria loading layout |
| `0x140303d97` | `VideoTask_RenderEditorScene` | DIVSS | editor render divisor |
| `0x140315c47` | `ShipCard_DrawBearingTrail` | MOVSS | ship-card trail length |

## How the tuning mod uses these findings

[hf-engines-and-rockets](https://github.com/Duumvir-hf/hf-engines-and-rockets) (`hf_tuning`) has
seven sites for its two `500.0f` scalar settings: one for `engine_overheat_threshold` and six
for `rocket_acceleration` (two thrust sites and four preview sites). The other 78 readers,
including the four enemy aim laser sites and the AI lead site, retain the vanilla pool value.

The [scalar installer](https://github.com/Duumvir-hf/hf-engines-and-rockets/blob/6f48fd7c7e42673b78fde3494cff9a2d1b12d51a/src/lib.rs)
rewrites the four displacement bytes directly; it does not write the original pool. If the
per-ammo acceleration hooks install, they replace the two thrust loads and use the configured
global acceleration as their fallback. The HUD preview still uses the global acceleration and
a fixed 1.5 s burn. A per-ammo acceleration override or a changed burn duration can therefore
make the preview differ from flight. The AI's 220 mm lead calculation keeps both vanilla values.

**Allocation pitfall (v0.2.0):** Windows rounds an explicit `VirtualAlloc` reservation address
down to its allocation granularity (64 KiB on the tested system). When a free region started at
the end of the image but not on a granularity boundary, requesting its first byte rounded back
into occupied memory. No displacement patches installed in the reported play test, although
the hooks still ran. The fix is to round the candidate address up and check that the whole
allocation still fits in the free region.

## Per-engine and per-ammo hooks

* **engine overheat per `m_oid`** — hook on `0x140028614` `MOVSS XMM0,[RSI+0x2ec]` in
  `Engine_ApplyThrustForce` (8 bytes), resume at `0x14002861c` (`COMISS`). For a matching `m_oid`
  (`Body+0x190`, `EscadraString`), the callback adds the difference between the configured global
  and per-engine thresholds to the temperature. This makes the native comparison implement the
  per-engine threshold when its operand uses the configured global value. If the scalar patch
  fails and `COMISS` still reads 500, the hook still runs but compares against the wrong baseline.
* **rocket acceleration per ammo `item_name`** — the rocket body carries **no** ammo identity:
  `FireShell`'s rocket cases build it inline (`New(0xa40)` + `Body_Constructor` + `CreateShell_`),
  never call `Body_InitializeCannonShell`, set no `+0x490` gun link and no `m_oid`; all they copy is
  `explosivePower`, `logic` (from `ammo->caliber`) and `ttl`, and NAR122 / 220 mm / NAR340 all share
  logic 130. The Ammo record is live at the four `body->logic = ammo->caliber` stores —
  `0x140030e90` (body RBX), `0x1400313c4` (RSI), `0x1400319d0` (RBX), `0x140031c46` (RSI), Ammo always
  in `R15`, six bytes `89 83/86 C4 03 00 00` — so a hook there stores `body → accel` in the mod's
  own map and the two burn sites (`0x1400348f2`, `0x140034c0c`) consult it. The thrust callback
  removes an entry when it sees a nonpositive burn timer; this is not a destruction hook.
* The 300 mm rocket (logic 132) can also be launched as a *ship part* (`creatureLogic` state 0); that
  path bypasses the four `FireShell` stores and does not supply an ammo override of its own.
* **burn time (v0.4.0)** — `Body +0x3d4` (`m_logic_f2`) is the burn timer, and the vanilla 1.5 s is
  stored in four places: `0x140030e9d` (`FireShell` rocket case, body RBX), `0x1400313d1` and
  `0x140031c53` (`FireShell` rocket cases, body RSI), and **`0x14003538e` in `creatureLogic` state 0 of
  logic 132** — the 300 mm rocket does *not* set its burn in `FireShell` (case 0x36 sets only ttl and
  the `+0x4a8` link), so that state-0 store is the one that covers it, for both the fired and the
  part-launched path. Each store is replaced by a callback writing the configured value; the rocket
  glow particle life is read from `+0x3d4` right after the store, so the visuals follow.
* Interoperability: the `FireShell` launch hooks do not overlap the known `ammo_extended` switch
  site at `0x1400302c8`. The trampolines install through patchy-rs, but the scalar displacement
  writes bypass its overlap tracking. This is not a blanket compatibility guarantee.
