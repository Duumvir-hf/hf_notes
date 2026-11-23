# Friendly-fire phasing: shells through allies, shells through enemies

Status: **assessment and patch plan; nothing implemented or tested in game.** 2026-11-05. Addresses are Steam 1.163 (`Highfleet.exe.unpacked.exe`, image base `0x140000000`) unless stated. Related: [weapons index](weapons-index.md) s6, [firing and shells](firing-and-shells.md), [blast and penetration](blast-and-penetration.md), [shell detonation paths](shell-detonation-paths.md), [Box2D physics setup](../physics/box2d-physics-setup.md) s5.

Goal: projectiles pass through ships on their own side instead of colliding with them; optionally, their blast does not damage same-side ships either. Symmetric by design: player shells pass through allies, enemy shells pass through other enemies. "Friendly fire off", not "self-damage off": a ship's own shells may still splash it, and ammo-rack chain reactions stay.

## Verdict

**Low-to-moderate difficulty, very feasible.** The game already implements the own-ship case; extending it to same-side is a focused filter patch, not a new system. The hard part is not discovery but covering the parallel damage/fuze paths and validating in game.

- Direct-hit-only prototype for cannon shells: **~1-2 days**.
- Solid feature (direct hits + no splash): **~1-1.5 weeks**.
- Full coverage (rockets/missiles/bomblets, proximity fuzes, KAZ, AI gate, config/polish): **~2 weeks**.

## What already exists

- **One collision gate.** `b2CFilter_ShouldCollide 0x1402858c0` is the entire filter. All fixtures use Box2D category 1 / mask `0xFFFF` / group 0, so the game's callback decides everything ([box2d-physics-setup.md](../physics/box2d-physics-setup.md) s5).
- **Own-ship immunity already works.** The filter returns 0 when both bodies share `Node.owner.id` (`Node.owner` at Body `+0xb0`: pointer `+0xb8`, id `+0xc0`), plus master/grandparent rules. [weapons-index.md](weapons-index.md) s6.C states: *"a shell never hits its own ship's parts; other ships (also allied ones) are hit."*
- **Shells already carry a side.** `Body::BulletImpact__ 0x140052a80`:
  - `SetLink(&this->base.owner, owner)` stores the firing creature (R8) in the owner link.
  - stores `alignment_override` at **Body `+0x248`**; `FireShell` passes `creature->m_alignment` (`Creature+0xad4`) when the argument is 0. This is the projectile's side id, and `b2CFilter_ShouldCollide` already reads `+0x248` for the same-side plane/bullet test via `SideIdsSameOrParent 0x140220120`.
  - stores `side` (`+-1`) at Body `+0x204`, but `+0x248` is the better value because it preserves the actual `m_alignment` (including faction ids).
- **AI already has a friendly-fire gate.** `AnyFriendlyCreatureWithinRadiusMul 0x14021c790` checks same-side creatures within a segment; called from `AiWeaponController` and `AircraftLogic_InsideFriendlyShipBubble`.
- **Hook pattern exists.** `highfleet-allies/qol/src/guns.rs` is a working code-cave example with `patchy::Trampoline`, including stolen-byte replay and register preservation.

## The desowning pair is not a fuse

`Node+0xf0` = `desowningTime`, `Node+0xf4` = `desowningTimer`.

- `Node_InitDefaults 0x1402804b0` sets `desowningTime = 0.75`, `desowningTimer = 0`.
- `Body::BulletImpact__` sets **both** to the ammo TTL (`this->desowningTimer = fVar10; this->desowningTime = fVar10;`).
- `b2CFilter_ShouldCollide` gates the owner/master/grandparent checks on:
  `timer == 0 || time - timer <= 3.0` for both bodies.
- The only routine seen to decrement/clear the pair is `0x140049c30` (vtable slot `+0x1d8`): it copies the owner link to `link2` (`+0x98`), decrements `+0xf4`, clears the owner when the timer reaches 0, and clears weapon state `+0x818` / `+0x820` / `+0x180`. Its exact call schedule is not proven; because it also clears weapon state, it is not the normal per-frame flight update.

Consequence: the own-ship protection is the **owner-id equality**, not a fuse. The desowning pair is bookkeeping that keeps the owner link alive. The shell sets both equal; if the routine is not called in flight (the likely case), the difference stays 0 and the owner check stays enabled. If it is called, the stock own-ship immunity would expire after 3 s — that is the open question below, and the side-filter plan is independent of the answer. Changing the pair will not protect allies; it can only affect whether the owner checks run for the owner ship. The `desowningTime - desowningTimer <= 3.0` clause is a grace for dropped/globalized bodies, not a detonation suppressor.

## Patch plan

### 1. Direct hits: `b2CFilter_ShouldCollide`

Inject at **`0x140285907`** (7 bytes, `MOV RDI,[R8+0x40]; TEST RSI,RSI`), jump to a code cave:

- At the site, `RSI` = game Body A, `R8` = fixture B (Body B at `[R8+0x40]`).
- If exactly one body is a cannon shell (`(meta & 0xd) == 0xd` at Body `+0x1c0`) and the other has an owner creature, compare sides:
  - shell side: `Body+0x248` (side id); fallback: `owner.node +0xad4` if `+0x248` is 0.
  - other side: `owner.node +0xad4`; fallback `Body+0x248`.
  - same-side test: the game's own form, `a != 0 && b != 0 && (a == b || a/100 == b || b/100 == a)` (`SideIdsSameOrParent 0x140220120`; the `/100` clause makes the normalized `+-1` tag match faction ids such as `100`).
- If same side and different owner id: jump to the `return 0` epilogue **`0x140285ff4`**.
- Otherwise replay `MOV RDI,[R8+0x40]` and `TEST RSI,RSI`, then resume at **`0x14028590e`**.

Null-check both bodies in the helper: the injection point is before the stock null checks. The helper must only read memory and must not allocate or panic.

This covers plain shells, canister pellets and fragments (`meta & 0xd == 0xd`). It deliberately leaves the native 50% shell-vs-shell rule and the missile/plane rules alone. Rockets/missiles (`meta` 0x15/0x25, logic 1/3/4/10/130/132/135/140/160) need a wider classifier; do not simply drop the shell-only test, because that would change shell-vs-shell and missile-vs-missile behaviour.

### 2. Splash: `Body_Explode` / `Body_AreaDamageFromBlast`

A collision-only patch is not full friendly-fire prevention: a shell that detonates near an ally still splashes it.

- `Body_Explode 0x140029b60` gathers every live body in radius with **no side filter**.
- `Body_AreaDamageFromBlast 0x14002cdc0` has **no side test**; the 0.66/1.0 factor is only a player-ship damage scale.
- `Body_Explode` also calls `Body_ApplyDamage` directly through vtable `+0x210` at **`0x14002a7a8`**, **`0x14002a822`** (direct-hit body) and **`0x14002a9ce`**, **`0x14002aace`** (area candidates). These bypass `Body_AreaDamageFromBlast`.

Recommended gating: skip **same-side, different-owner** victims, keep same-ship self-damage.

- Area call: inject at **`0x14002ce10`** (7 bytes, `MOVSD XMM0,[RCX+0x210]`), after the `*param_6 = 0` store at `0x14002ce0d`. At this point `RCX` = victim, `RBP` = exploder. If same-side different owner, jump to the epilogue **`0x14002d02d`**; otherwise replay the `MOVSD` and resume at `0x14002ce18`.
- Direct calls: gate the four `CALL [RAX+0x210]` sites with the same helper. Victim is in `RCX` (`R15` or `RBX` depending on the site), exploder is in `RBP` at all four sites. If blocked, skip the call and continue after it.
- Owner identity: `Body+0xc0` is the owner id (ship id) and `Body+0xb8` the owner creature. For a shell, `+0xc0` is the firing ship id; for a ship part, its ship id. Compare ids first (same ship -> allow), then sides.
- Missiles may have a cleared owner link; fall back to the separate launch-owner link at `Body+0x4b0` for logic 1/3/4/10.

Keeping same-side bodies in the candidate list preserves the blast shadow/occlusion model: an ally still shields an enemy behind it, but takes no hull damage. That is a reasonable "friendly fire off" behaviour.

### 3. Proximity fuzes

Even with collision and splash gated, a fuze can still detonate on a friend:

- Logic **160** `_PROXY` (state 2) scans bodies by `owner.id` (`+0xc0`), distance <= 150 and a 36-degree cone. It will fuze on allies ([weapons-index.md](weapons-index.md) s5.5).
- Logic **132** 300 mm rocket fuze scans by owner id, radius >= 20, distance <= 500, closest approach. It will fuze on allies (s5.3).
- Logic **130** rocket burst already skips same-side via `side * side > 0` (s5.2).
- Logic **190** bomblet uses the owner's side list (s5.6); logic **140** LG picks enemies by side (s5.4).
- Missiles 1/3/4 need an audit of their proximity fuse and retarget scans.

Patch logic 160 and 132 to add a side check (or replace the owner-id test with the side test). This is not damage prevention; without it a friendly ship can still waste a shell by fuzing it early.

### 4. Optional: KAZ and AI gate

- `Weapon_FlakInterceptScan 0x14002f480` excludes the KAZ's own `owner.id` (`param_1[0x17]` / `param_1[0x18]`) but not other ships on the same side. An ally's KAZ can intercept the player's shells. Add a side check if that matters.
- `AnyFriendlyCreatureWithinRadiusMul 0x14021c790` makes the AI hold fire when a friend is in the line. Safe, but the AI will not exploit phasing. Patch only the two `AiWeaponController` call sites if wanted; the function is also used by aircraft logic.

## Effort

| Scope | Work | Estimate |
|---|---|---|
| Direct hits, cannon shells | `ShouldCollide` cave | 1-2 days |
| + splash immunity | 1 area + 4 direct damage sites | +1-2 days |
| + proxy/rocket fuzes | logic 160/132 + missile audit | +2-3 days |
| + KAZ/AI gate | optional | +1-2 days |
| Regression/balance | in-game tests | +1-2 days |
| **MVP** | direct hits only | **~1-2 days** |
| **Solid feature** | direct + splash + fuzes | **~1-1.5 weeks** |
| **Full polish** | all projectiles, AI, KAZ, config | **~2 weeks** |

## Risks and edge cases

- **Global filter.** A bad patch can make hulls non-collidable or shells inert. Byte-check the site, guard every read, and put the feature behind a config flag (default off).
- **Shell classification.** `meta & 0xd == 0xd` is the game's own "projectile" test; it covers cannon shells, canister and fragments, not rockets/missiles. Do not remove the native 50% shell-vs-shell rule or the missile/plane rules by accident.
- **Multiple blast paths.** Patching only `Body_AreaDamageFromBlast` leaves the four direct `Body_ApplyDamage` calls.
- **Owner lifetime.** Owner links can be null or cleared; null-check. Missiles may need `+0x4b0`.
- **Neutral side 0.** Native targeting treats 0 as hostile (`side_self * side_other <= 0`). The both-non-zero `SideIdsSameOrParent` form keeps neutrals hittable.
- **Hierarchical side ids.** Use the game's `/100` comparison, not a plain sign test, if faction ids are used.
- **Balance.** Enemies stop damaging each other; allies become much more durable. This is a significant buff for the player's side.
- **In-game validation.** No regression tests were added (repo policy). Verify with a probe/log first, then in game; record what remains untested.

## Validation checklist

1. Player fires across an ally at an enemy: shell passes through the ally, hits the enemy, ally takes no damage.
2. Enemy fires across another enemy at the player: shell passes through the enemy, hits the player, the enemy ally takes no damage.
3. Shell detonates near an ally: no splash damage to the ally; the firing ship's own splash still damages itself.
4. Ship-vs-ship collisions still work; hulls do not phase.
5. Ground and neutral targets are still hit.
6. Proxy and 300 mm rocket fuzes do not trigger on friends.
7. KAZ still intercepts enemy ordnance; friendly ordnance is not intercepted (if patched).
8. Large battle frame time unchanged.
9. Co-op/multiple player slots: all positive-side ships are covered.

## Open questions

- Does `desowningTimer` (`+0xf4`) decrement on a shell in flight, or only on the drop/teardown path? A breakpoint on a live shell settles it. If it decrements, the stock own-ship immunity is time-limited to 3 s, but the side patch is independent.
- Are there other `creatureLogic` blocks that scan by owner id rather than side? Audit 130/132/135/140/160/190 and missile ids 1/3/4/10.
- Is `Body+0x248` set for every ammo type, including canister and rockets, or only the `BulletImpact__` path? `FireShell` writes it explicitly for some cases; verify the rest.
- Does a ship changing side mid-battle change the behaviour of shells already in flight? The patch reads sides live, so it should; test with a captured/prize ship if possible.
- Do any level scripts rely on friendly ships damaging each other?

## References

- [weapons index](weapons-index.md) s6.C (own ship vs allies), s5.2-5.6 (projectile logic scans).
- [firing and shells](firing-and-shells.md) (`BulletImpact__`, owner, TTL).
- [shell detonation paths](shell-detonation-paths.md) (arming and contact rules).
- [blast and penetration](blast-and-penetration.md) and [explosion damage 1.151 vs 1.163](explosion-damage-1151-vs-1163.md) (blast model).
- [Box2D physics setup](../physics/box2d-physics-setup.md) s5 (filter rules).
- `highfleet-allies/qol/src/guns.rs` (code-cave pattern).
