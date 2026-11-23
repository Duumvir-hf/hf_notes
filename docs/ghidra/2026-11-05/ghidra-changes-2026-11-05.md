# Ghidra changes, 2026-11-05 (friendly-fire phasing assessment)

Program: `Highfleet.exe.unpacked.exe` v1.163 (project `HighfleetServer`),
image base `0x140000000`. This pass documents the friend/foe projectile
question in [friendly-fire-phasing.md](../../findings/combat/friendly-fire-phasing.md).
No code bytes were changed; only names and comments.

## Rename

| address | old name | new name | evidence |
|---|---|---|---|
| `0x140049c30` | `rep_1400477f0_MakeGlobal?` | `Node_UpdateDesowning` | Body/Frame/Creature vtable slot `+0x1d8`; copies the owner Link (`+0xb0`) to `link2` (`+0x98`), decrements `desowningTimer` (`+0xf4`) and clears the owner when it reaches 0, then clears `+0x818` / `+0x820` / `+0x180`. The old 1.151 label is stale: `0x1400477f0` is not a function entry (rep audit 2026-10-29). |

The old name is preserved in the plate comment. The name is **behavioural**,
not a proven identity; the rep audit still lists the 1.151 twin as unknown.

## Plate comments

| address | function | addition |
|---|---|---|
| `0x1402858c0` | `b2CFilter_ShouldCollide` | Clarified that the early owner check is `Node.owner.id` (`+0xc0`), not a separate group; documented the `desowningTimer`/`desowningTime` gate; documented the shell side id at Body `+0x248`; recorded the same-side phasing hook candidate (`0x140285907`, resume `0x14028590e`, return-0 `0x140285ff4`). |
| `0x140029b60` | `Body_Explode` | No side/friend filter; damage is applied both by `Body_AreaDamageFromBlast` and by direct `Body_ApplyDamage` calls via vtable `+0x210` at `0x14002a7a8`, `0x14002a822`, `0x14002a9ce`, `0x14002aace`. |
| `0x14002cdc0` | `Body_AreaDamageFromBlast` | The 0.66/1.0 factor is only a player-ship damage scale, not a side test; gate candidate `0x14002ce10`, early-out `0x14002d02d`. |
| `0x140052a80` | `BulletImpact??` | Owner link = firing creature; side id at `+0x248`; `desowningTime`/`desowningTimer` = TTL; the pair is owner-link bookkeeping, not a fuse. |
| `0x14002f480` | `Weapon_FlakInterceptScan` | Excludes its own `owner.id` only; an ally's KAZ can intercept friendly shells. |
| `0x14021c790` | `AnyFriendlyCreatureWithinRadiusMul` | Segment/radius/side semantics; callers; it is the AI's "do not shoot through a friend" gate, not a collision rule. |
| `0x140220120` | `SideIdsSameOrParent` | The `/100` faction-id clause and where the test is used. |

## EOL comments

| address | note |
|---|---|
| `0x140285907` | Same-side phasing hook candidate in `ShouldCollide` (7 stealable bytes, replay/resume addresses). |
| `0x140285de7` | `SideIdsSameOrParent(Body+0x248)` covers planes/bullets only, not cannon shells vs ships. |
| `0x14005306d` | `Body+0x248` = projectile side id. |
| `0x140052b1e`, `0x140052b26` | `desowningTimer` / `desowningTime` = ammo TTL, not a fuse. |
| `0x14002ce10` | Blast friendly-fire gate candidate in `Body_AreaDamageFromBlast`. |
| `0x14002a7a8`, `0x14002a822`, `0x14002a9ce`, `0x14002aace` | Direct `Body_ApplyDamage` calls that bypass the area function. |

## Caveats

- No implementation and no in-game validation. The patch plan is a reading of
  the 1.163 decompile, not a tested change.
- Whether `desowningTimer` decrements on a shell in flight is still open; a
  breakpoint on Body `+0xf4` would settle it. The side-filter plan does not
  depend on the answer.
