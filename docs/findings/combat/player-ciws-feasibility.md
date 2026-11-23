# Player-side autonomous CIWS for 37 mm: feasibility

Status: **feasibility study + v0.1.1 prototype; v0.1.1 validated in game 2026-11-06** (the 37 mm tracked and fired at `MDL_MISSILE_01`). The full session record, architecture, native notes, testing and evidence are in `hf_ciws/docs/`. Addresses are Steam 1.163, image base `0x140000000`. Tags: `[code]` read in the decompile/disassembly, `[data]` read from `OL.seria` / ship files, `[proposal]` design choice, `[unknown]` needs a live probe.

**Verdict: feasible without an engine change.** The game already has the three hard pieces:

1. **Per-part gun servos.** Command 200 carries one angle to one gun part; every turret has its own revolute joint and motor. The engine can rotate each 37 mm independently today.
2. **Per-gun firing.** `FireGuns(gun)` (`0x1400559e0`) takes one gun Body, not a ship or a group. Autonomous fire does not need the player's LMB/RMB bytes.
3. **AI CIWS logic.** The stock AI already aims and fires 37 mm at missiles and aircraft (`AiWeaponController 0x14007a8d0`, threat list `0x1400625a0`, chooser `0x14006c100`). The player's gun commands come from the player path instead; whether the AI controller is also entered on the controlled ship is open (section 1).

The missing piece is a **player-side controller**, not a new weapon system. A new "module" part is data-only and acts as a marker/cost item; the behaviour has to come from a DLL.

## 1. What is already there

### AI CIWS works, but only on AI ships

`AiWeaponController` aims every gun at the ship's missile/aircraft target and calls `FireGuns` when the native range, alignment, friendly-fire and reload gates pass. It is called from `AiShipTick` for AI-flown ships.

The player's ship is not AI-flown (`+0xbd4 == 0` while controlled), and its gun commands come from `UpdateShipsAndMapFrame`. **Open:** the ship is registered in the AI vector before `SetControlled` clears `+0xbd4`, and nothing read so far removes it; whether `aiShipDriver` still enters it is unresolved (`../physics/player-attitude-stabiliser.md`). In play the player's manual command wins, but a log-only build should record whether `AiWeaponController` or `AiShipTick` is entered with the player's pointer. The controller design below does not depend on the answer: the call-site hook replaces the player's command and fires explicitly.

This is also the proof that 37 mm can kill missiles in this engine: it is the vanilla CIWS weapon.

### The player's sender uses one angle for both groups

`UpdateShipsAndMapFrame 0x1402a9940` contains the player's gun command loops:

| Loop | List | Fire button | `Part_HandleCommand` call |
|---|---|---|---|
| group 0 | `creature+0x13a8..+0x13b0` | LMB `+0xbbc` | `0x1402aa351` |
| group 1 | `creature+0x13c0..+0x13c8` | RMB `+0xbbd` | `0x1402aa380` |

Both loops build the same stack command record (`code = 200` at `+0`, angle at `+4`) and send `ship+0xbb4 - pi` to every gun. The angle is the hull/mouse aim angle; the subtraction constant is `pi` at `0x14035280c` `[code]`.

That is the only reason the guns "share rotation": one high-level command is sent to all of them. The servo itself is per part.

### Weapon groups already exist, and the group field is `m_weapon_type`

`Creature_AccumulatePartStats` reads `part+0x84c` and compares it with 0/1:

```asm
1400718da  MOV EAX, dword ptr [RSI + 0x84c]   ; m_weapon_type
1400718e0  TEST EAX, EAX
1400718e2  JNZ 0x1400718fd
...        ; group 0 -> creature+0x13a8, reach +0x1838
1400718fd  CMP EAX, 1
...        ; group 1 -> creature+0x13c0, reach +0x183c
```

`m_weapon_type` is serialized (`Body::SerializeHeader`, `+0x84c`). Vanilla ship files leave it absent, so every vanilla gun is group 0. Setting `m_weapon_type=1` on the 37 mm turrets in a ship design puts them in the RMB/AA group, with their own reach `+0x183c` and their own pre-combat ammo preset (`EDC_TEST_AMMO2_*`) `[code/data]`.

**Correction to the older note:** `ai-telemetry.md` says the group is the part field at `+0x7a4`. That is not the gun group. `+0x7a4` is a byte initialised to 0 by `setCertainBodyFloatVals` and read by GUI code; the gun group is `m_weapon_type` at `+0x84c`. The assembly above is definitive.

Why group 1 matters for CIWS: the servo at `0x140053870` applies **parallax correction** when the owner has a target link. It picks the link in this order:

* if `owner+0xb00` (ship target) is non-null, use it;
* otherwise use `owner+0xb18` (missile/aircraft target);
* additionally, a part with `m_weapon_type == 1` uses `owner+0xb18` whenever that link is non-null, even if `+0xb00` is also set.

For a 37 mm turret at 400 units, a 50-unit mount offset is about 7 degrees of parallax. If the 37 mm stays in group 0 and the player also has a ship target, the correction follows the ship target, not the missile, and long-range CIWS shots will miss. Group 0 happens to use `+0xb18` when the player has no ship target at all, which is why a pure missile-defence scene can appear to work; a mixed battle will not.

The v0.1.0 prototype works around this without a design edit: it temporarily sets the part's `m_weapon_type = 1` around the native `Part_HandleCommand` call, so the servo takes the `+0xb18` branch for a group-0 37 mm too, then restores the field. The design's fire group is unchanged. A real `m_weapon_type=1` in the design is still useful for a separate RMB fire group, reach and pre-combat ammo, but it is no longer required for CIWS parallax.

### 37 mm and FCR numbers

`[data]` from `Libraries/OL.seria`:

| Part | Caliber key | Magazine | Reload | Rate | Power | Crew |
|---|---|---|---|---|---|---|
| `MDL_CANNON_30_6` | 5 (37 mm) | 50 | 20 s | 2000 rpm | 750 | 6 |
| `MDL_FCR_01` | tracking 400, radar 400, category 65536, sector type 2 | | | | 2000 | 8 |
| `MDL_FCR_02` | tracking 250, radar 250, category 65536, sector type 2 | | | | 1000 | 6 |

The 37 mm reach contributed to the ship is 800 (`Creature_AccumulatePartStats`, case 5). The FCR tracking is 400/250, so **the FCR is the limiting range**, exactly as the idea asks.

`MDL_FCR_01/02` are the only vanilla parts with `m_mdl_tracking != 0`; radars have radar but no tracking. So "requires FCR" can be implemented as "requires a live, powered part with `Stats+0x110 > 0`" and will match vanilla semantics.

The FCR's sector type 2 is a full 360-degree scan with line-of-sight blocking, not a narrow cone. The vanilla "tracking limitation" is therefore **range, blockage/LOS, power, damage and destruction**, not a narrow arc. A modded FCR with a narrower sector would be handled by the same sector array.

## 2. The controller

### Hook shape

Two five-byte call-site hooks in `UpdateShipsAndMapFrame` are enough:

* `0x1402aa351` (group 0)
* `0x1402aa380` (group 1)

Use `patchy`'s `patch_call` with `save_overwritten = true`: the trampoline runs our callback first, then executes the original `Part_HandleCommand` call with the modified command. The callback receives `RCX = part`, `RDX = command record`; it may write the angle at `[rdx+4]` and let the original call send it. No global `Part_HandleCommand` detour is needed, so engines, legs and AI commands are untouched.

The callback should:

1. Read the gun's `part+0xb8` owner; accept only a player-controlled creature (`+0xbd4 == 0`, side > 0, state not 6/7).
2. Read `GetStats(part+0x188)` (`0x140281e00`) and require `Stats+0x144 == 5` (37 mm). Other calibres pass through.
3. Check the ship has a live, powered, enabled **module** (config oid list) and a live, powered, enabled **FCR** (tracking > 0). If not, pass through.
4. If a CIWS target exists, write the CIWS command angle into `[rdx+4]` and set the ship's missile target link `+0xb10` to the target (so the servo's parallax correction uses it for group 1).
5. If no target, leave the command exactly as the player sent it.

This gives the effect the idea asks for: group-1 37 mm rotate to the CIWS target while the main guns keep following the mouse. The player's RMB still fires the group manually; the controller also calls `FireGuns` itself.

### Target scan and fire

Per ship, at about 10 Hz (cache the result; aim/fire every frame):

* Walk the flat body array `0x143a33c90`, count at `0x143a51150`, cap 15000 (the same array `Weapon_FlakInterceptScan` and hf_ally's plane code use).
* Keep active bodies with category bit `0x4000` (tactical missile), `0x400` (aircraft) or `0x8000000` (strategic launch). `checkShipPartFlagActive` is the native test, but a direct `Stats+0xb0` read plus the active/alive test is enough.
* Reject same-side bodies. Read `body+0x248` (side) first; if 0, use the owner link `+0x4b0` (the hf_ally `owner_side` helper). This avoids the stock AI's pass-1 friendly-fire bug where a modded strategic-launch body is engaged by its own side.
* Require the body to be closing on the ship: `(ship_pos - body_pos) · body_velocity > 0`, and time-to-closest-approach within a configurable horizon. This avoids shooting missiles flying away.
* Require the target to be inside the FCR's tracking coverage (section 3).
* Require a valid shell solution: use the game's plain-shell quadratic (`|V|^2 - s^2)t^2 + 2(P·V)t + |P|^2`, smallest positive root, as in `AiWeaponController`/`0x1402333c0`). `s` comes from `AMMO_LIST[gun+0x850].speed` (`AMMO_LIST_BEGIN 0x143a13be0`, stride 0x188, speed at +0x158). Reject if the flight time exceeds the ammo TTL (+0x16c; 37 mm is 2.0 s) or the aim point is impossible.
* Pick the most imminent valid target (smallest positive flight time, or distance as a tie-break). One target for all 37 mm is enough for v1; later, distribute guns over targets.

Aim and fire, per gun:

* Command angle = `-atan2(-dx, -dy) - pi`, where `(dx,dy)` is the ship-centre-to-aim-point direction. This is the same convention the player sender and the servo use (the servo adds `pi` and uses `(sin, -cos)`).
* Set `ship+0xb10` to the target with `SetLink 0x140194de0`. The servo then computes the gun-local angle from the gun's own position to the point on that ray at the target's distance. That is the native parallax correction.
* For the firing check, compute the same gun-local desired angle from the gun position (`part+0x128/+0x12c`) to that point, and compare it with `part+0x138` modulo 2π. Tolerance 0.025-0.03 rad matches the AI.
* If aligned and inside the burst window, call `FireGuns(part)` (`0x1400559e0`). The native reload (`+0x860` loaded, `+0x86c` rate timer) and power gates still apply, so no custom rate limiter is needed. Add a burst/pause (e.g. 0.3-0.4 s fire, 0.4-0.6 s pause) so a 50-round magazine is not dumped into one missile.
* Call the native friendly-fire test `AnyFriendlyCreatureWithinRadiusMul 0x14021c790` with the ship position, a 2000-unit segment along the aim direction, side +1, radius factor 3.0 and the ship as the excluded shooter. The AI uses exactly this gate; the player's native fire path does not.

The AI's `AiWeaponController` can be used as a reference for the aim point and gate order; it is not called on the player's ship.

## 3. FCR / tracking gating

Two levels, cheapest first:

1. **Require an FCR.** At least one part in `creature.parts` (`+0x1330`) with `Stats+0x110 > 0`, `part+0x48 != 0`, `part+0x80c != 0` (enabled), `part+0x818 != 0` (powered), health `+0x2c8 > 0`. If none, no CIWS. If the FCR is destroyed or unpowered, CIWS stops.
2. **Use the ship's merged tracking coverage.** The stats pass merges every tracking part into `creature+0x16c8` (the repaired twin is `+0x1740`), 360 floats, max rule. Only parts with `m_mdl_tracking != 0` call the merge, so radars do not contaminate this array. The value is the effective range per direction, already scaled by damage and zeroed where the hull blocks the sensor. Use:
   * `bearing = atan2(target_dy, target_dx)`;
   * if the array is ship-local, `index = wrap(bearing - ship+0x138)`; if it is world-oriented, `index = wrap(bearing)`;
   * accept only if `distance <= array[index]` and `array[index] > 0`.
   If the array is empty or the orientation probe fails, fall back to `distance <= max(FCR Stats+0x110) * range_scale`. The fallback still respects the FCR range but loses LOS/arc.

Vanilla FCR is 360 degrees, so the fallback is not a big loss. The array is worth using because it gives the "same tracking limitations" for free: hull blockage, damage scaling, and any future narrow-sector FCR.

`[unknown]` **Orientation probe needed:** `Body_ScanSectors 0x140052340` fills `Body+0x508` (360 floats) and `Creature_MergeSectorArray 0x140061260` copies it into the ship arrays without an angle argument. That suggests the part array is in the part/ship local frame, and the index should subtract the ship angle. Confirm by logging `trackingSectors` on a ship with one FCR at two hull headings: if the pattern stays at the same index relative to the hull, it is local; if it follows world north, it is world. Until then, use the range-only fallback.

Multiple FCRs merge by max, not sum (same as the native rule). Two FCR_02s do not make 500; the module must not add them.

## 4. The module part

A new part is **data only**. Nothing in `OL.seria` or `parts.seria` can make a part run code; the game has no per-part script. The DLL identifies the module by `m_oid` and enables the controller. The module is therefore a gate/cost item, and its destruction/power loss can disable CIWS.

Suggested data shape `[proposal]`:

* `OL.seria` Stats: `m_oid=MDL_CIWS_01`, a name like "CIWS FIRE CONTROL", `m_category=65536` (sensor/computer) or 64 (block), `m_mdl_power_need=300`, `m_mdl_power_priority=1`, `m_mdl_crew_need=2`, `m_mdl_armor=1`, price 2000-4000, **no radar, no tracking**, `m_mdl_sectors_type=0` (it does not scan). The DLL only reads the oid, so the category is a UI/balance choice.
* `parts.seria` Body: clone a small sensor/block body; keep a valid `m_slot_type` (the FCR uses slot type 5), a valid mesh/sprite, low mass. `m_logic=0` so the part is inert; the DLL does the work.
* Language keys `MDL_CIWS_01`, `_SDESC`, `_DESC`, `_KEY` in both language packs. The recipe and encryption tool are in `hf_aa_missile/NOTES.md`.
* `Modloader/config/Shipworks.json`: add `"MDL_CIWS_01"` under `"Sensors"` (or `"Defences"` if the body uses that slot type).
* Campaign availability: qol `shop_parts` or a custom starting ship.

The game rewrites `OL.seria` and `parts.seria` at launch, so edit them with the game closed and keep backups. The existing data recipe in `hf_aa_missile/NOTES.md` covers stats, body, names, palette entry and the revert procedure.

The module should require **both** a healthy powered copy of itself and a healthy powered FCR. One module per ship controls all 37 mm (caliber 5); multiple modules do not stack. The FCR itself is the range limiter; the module does not extend it.

## 5. Shipworks Extended: what is actually needed

The belief "we need Shipworks Extended" is **half right**:

* You need **ShipworksExtended installed** (or another palette injector) to see a **new** part in the Shipworks test editor.
* You do **not** need to modify or rebuild ShipworksExtended. Its `Modloader/config/Shipworks.json` is a name list per palette category; it calls `DefinePart` for each name. Add the new oid to the `Sensors` array and restart the editor. The mod reads the config each time the Shipworks is entered.
* A bad or missing oid crashes the Shipworks (the mod's own log says so). Clone a known-good part and keep the name identical in `OL.seria`, `parts.seria` and `Shipworks.json`.
* The palette is only needed to **place** the part by hand. A part already present in a ship design's `.seria` loads without a palette entry; direct design editing is a valid fallback if the palette entry is inconvenient.
* For the **first behaviour test, no new part is needed at all.** Many vanilla ships already carry both an FCR and 37 mm: `Fenek`, `Fenek Mk2`, `Gepard`, `Gepard Mk2`, `Kormoran`, `Negev`, `Egida`, `Egida Mk2`, `Triumphant`, `Varyag`, `Vega`, `Nomad`, `Novorossiysk`, `Typhon`, `Voskhod X2` (counted from `hf_ml/data/Ships_seria`). Enable CIWS for any ship with an FCR (`module_oids: []`), put its 37 mm in group 1 in the test design, and test the behaviour before doing the data work.
* The Shipworks test fight is `EDC_TEST_FIGHT` (`ClickFunc 0x14013d7b7`): it loads `Objects/Designs/Test/<id>.seria` into a player escadra, builds the enemy fleet, sets `DAT_147fc6fb6 = 1`, and starts `Combat.seria`. So the editor can produce a real fight with enemies; confirm that the selected enemy group includes missiles. If it does not, use a campaign battle against a missile-heavy enemy or the strategic-missile defence scene.
* For the module test, the test design can include FCR + module + 37 mm; the vanilla palette already has the FCR and 37 mm, so only the module needs the new palette entry.

## 6. Prototype plan

1. **Log-only hook.** Hook the two call sites; log ship, part, caliber, group, FCR parts (oid, tracking, power, health), module parts, and the player's command angle. No behaviour change. Confirm the hook sites and part identification in a Shipworks test fight.
2. **Fixed-offset aim.** Replace the 37 mm command with `player_angle + 30 degrees` and confirm the 37 mm rotate independently while the main guns follow the mouse. This proves the per-part override and the angle convention without needing threats. Also confirm the command angle sign by aiming at a known direction.
3. **Threat scan log.** Scan the body array at 10 Hz; log hostile missiles/aircraft inside the FCR range, their side/owner, closing test, and the chosen target. No fire. Validate the side/owner logic against the player's own launches.
4. **Aim + fire.** Enable the controller with the native friendly-fire gate and a burst limit. Test in a missile scenario/test fight. Measure rounds used, hits, and whether the 37 mm ever fire through a friendly.
5. **FCR gating.** Unpower or destroy the FCR in the editor/battle; confirm CIWS stops. Repeat with the module.
6. **Module part.** Add `MDL_CIWS_01` to the data files, language, and `Shipworks.json`; test with and without the module.
7. **Balance.** Tune `range_scale`, burst/cooldown, target horizon, and module cost/power/crew. Compare with the stock AI's 37 mm CIWS.

## 7. Risks and open questions

* **`trackingSectors` orientation** (`+0x16c8`): local vs world. Probe with two hull headings; fall back to range-only until known.
* **AI controller on the player's ship.** If `aiShipDriver` still enters the controlled ship, `AiWeaponController` may command and fire its guns in parallel with the player's path. In play the player's command wins, but the AI could add `FireGuns` calls. The log-only build must record whether `AiWeaponController`/`AiShipTick` is entered with the player's pointer. If it is, suppress the AI controller for the player's ship (a guarded one-line return) or accept only the player-path fire.
* **Group 0 37 mm.** The prototype temporarily sets `m_weapon_type=1` around the native command, so the servo's parallax correction uses the missile target `+0xb18` for group 0 too, then restores the field. The residual risk is that the temporary write is visible to any other handler in that one `Part_HandleCommand` call; the command code is 200 and the servo is the handler that reads the field. A design-level `m_weapon_type=1` removes the temporary write and gives a separate RMB group, but is not required for the prototype.
* **`+0xb10` side effects.** Setting the player's missile/aircraft target link every frame may change the HUD lock marker and any system that reads `+0xb18`. The link is already the player's "missile target" link, so this is semantically correct, but it needs a play test.
* **Unknown-owner missiles.** A strategic missile whose side/owner cannot be resolved is the friendly-fire case that produced the `intercept_ignore_oids` side patch. Default to engaging only resolvable hostile owners, or provide an oid ignore list; do not blindly copy the AI's pass-1 "no side test".
* **37 mm effectiveness.** 50 rounds, 20 s reload, ±0.05 rad spread. At 400 units the spread is about 20 units; CIWS is probabilistic. Burst length and multiple guns matter. A modded flak round (FireShell cases 60/70) is a possible later enhancement, not required.
* **Own-ship line.** `AnyFriendlyCreatureWithinRadiusMul` excludes the shooter, so it does not test the player's own hull. The turret joint limits and the game's collision filter are the only protection against firing across the deck. Watch for self-hits in the first live test.
* **Ammo selection.** Group 1 has its own pre-combat ammo preset. A wrong preset can load an ammo the 37 mm does not expect; test with the design's default 37 mm ammo first.
* **No in-game test yet.** Everything above is static. The first log-only build is the cheapest way to falsify the hook/angle/part assumptions.

## 8. Effort

* Log-only hook + part identification: 1-2 days.
* Threat scan + lead solver + per-part aim: 2-4 days.
* Firing, friendly-fire, burst control: 1-2 days.
* Module data + palette + language: 1 day.
* Play-testing and tuning: 1-3 weeks, depending on how much of the 37 mm accuracy/ammo behaviour needs adjusting.

A small standalone DLL is cleanest for testers (`hf_ciws`), but adding a `player_ciws` block to `hf_ai` is also viable: hf_ai already has `patchy`, the safe `rd` helpers, highfleet-rs layouts and logging. A standalone DLL avoids coupling the feature to hf_ai's tactical changes and can be disabled independently.

## 9. Recommendation

Build the prototype **without a new part first** on a vanilla FCR + 37 mm ship. Group 1 is optional: the prototype temporarily marks the 37 mm as `m_weapon_type=1` for the servo call, so group 0 also gets the missile-target parallax; a design-level group 1 is still useful for a separate RMB group, reach and ammo. Hook the two player command call sites, log first, then add the threat scan and fire. Add `MDL_CIWS_01` only after the behaviour works. ShipworksExtended can stay untouched; the module is one more name in `Shipworks.json`.

## 10. Prototype v0.1.0/v0.1.1 (built 2026-11-05; v0.1.0 log-tested in game)

The standalone `hf_ciws` DLL is at `hf_ciws/` in the workspace; the tester folders
are `hf_ciws/release/hf_ciws_v0.1.0/` and `hf_ciws/release/hf_ciws_v0.1.1/`. It
implements the first gate exactly:

* player-controlled ship (`side > 0`, `+0xbd4 == 0`, live driver state);
* at least one `m_weapon_caliber = 5` gun;
* at least one live, powered, enabled part with `m_mdl_tracking > 0`;
* no module part (`module_oids` empty) and no FCR sector/LOS mask yet.

It hooks the two call sites (`0x1402aa351`, `0x1402aa380`), temporarily sets the
37 mm's `m_weapon_type = 1` around the native `Part_HandleCommand` so the servo's
parallax correction uses `+0xb18`, sets that link to the chosen target, scans the
flat body array at 5 Hz, computes the native plain-shell lead, and calls
`FireGuns` when the barrel is aligned and the friendly-creature line test is
clear. Modes: `log`, `aim`, `fire`; burst/cooldown; configurable calibres, FCR
oids, module oids and ignore oids. The first live run should start in `log` mode,
then `aim`, then `fire`.

**First live log run, 2026-11-05 (game `Highfleet_2026-11-05_23-46-15.log`):**
the Modloader found and loaded `hf_ciws_v0.1.0.dll`, the version check passed and
both player call sites installed (`Installed 2 permanent patch(es)`). The gate
found player ships with 4 x 37 mm + FCR_02 (tracking 250), 2 x 37 mm + FCR_01
(tracking 400) and 1 x 37 mm + FCR_01, including both all-group-1 and all-group-0
37 mm layouts. Target scans found `MDL_MISSILE_01` (`cat=0x4000`, `side=-1.0`) at
shell flight times `t=0.19..0.31`; no `hf_ciws` error line appeared. The run was
in `log` mode, so nothing aimed or fired, and the flagship died in both test
battles. The first target line in each engagement was already at `t≈0.2-0.3 s`,
so the 37 mm turrets get a very short window against a fast Zenith, especially
with FCR_02's 250 range; a 10 Hz target scan was enabled for the next run. The
next test is `fire` mode with the aim-error log visible.

**v0.1.1 fixes (built 2026-11-05, test pending):**
* **One-shell burst bug.** In v0.1.0 the cooldown check ran before the in-burst
  check: the first shot set `cooldown_end = burst_end + 0.5`, so every later
  frame in the same burst returned "do not fire". The 23:52 fire log confirms
  the symptom (one `CIWS fire` line per target, then the target was gone). The
  check order is fixed; a burst now fires every frame until `burst_seconds`,
  with `+0x86c` as the shot-rate limiter.
* **Longer range.** `range_scale` is 2.0 by default: FCR_01 400 -> 800,
  FCR_02 250 -> 500, still FCR-gated. `CIWS parts` prints the effective range.
* **Burst defaults** are 1.6 s (a full 50-round 37 mm magazine) and 0.3 s pause.
* **Sync diagnostics.** `CIWS aim` now prints `cmd_in` (the player command before
  the hook), `cmd_out` (the CIWS command), `gun_ang` (the barrel's actual angle)
  and `group`. If `cmd_in` and `cmd_out` differ but `gun_ang` follows `cmd_in`,
  another sender is still driving the turret; if `gun_ang` follows `cmd_out`, the
  37 mm is independent and the earlier "sync" was the player aiming at the same
  missile or a visual turret-base effect.

**v0.1.1 in-game validation, 2026-11-06 (`Highfleet_2026-11-06_00-01-00.log`):**
it worked. `ship=0x190c4d10` had 2 x 37 mm group 1, FCR_01 tracking 400 and
effective range 800. The first target `MDL_MISSILE_01` was at distance 796,
`t=0.62`. The first gun's player command was `cmd_in=-4.771`, the CIWS command
`cmd_out=-2.179`, and the barrel went `-4.777 -> -2.953 -> -2.169 -> -2.114`
with `err` `2.601 -> 0.791 -> 0.030 -> 0.006`; both guns fired at range 363 and
the target disappeared. A second battle repeated at distance 795, `t=0.63`,
`err=0.007`, fire at range 371. The user confirms the CIWS worked. The flagship
was still lost in both battles; the CIWS only engages missiles/aircraft.

**New bug found in the same log:** the player sender reuses one command record
for the whole gun loop. The second 37 mm's `cmd_in` was exactly the first 37 mm's
`cmd_out` (`-2.179` / `-1.430`). That is the real mixed-group sync mechanism: a
main gun later in the same group would inherit the CIWS angle. v0.1.2 must restore
`[cmd+4]` after the native call. The `CIWS burst` count log also did not appear
because `prepare` updates `target_body` before `after` sees the change; the burst
itself fires. Full session docs are in `hf_ciws/docs/` (session report,
architecture, native notes, testing, roadmap and evidence).
